# komal-kashyap-project
AI based delivery identification in post offices
import os
import json
import streamlit as st
from PIL import Image
# AI generated need testing
from src.ocr_engine import OCREngine
from src.address_parser import AddressParser
st.set_page_config(layout="wide", page_title="OCR Address Geocoder")
@st.cache_resource
def load_engines():
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
model_path = os.path.join(BASE_DIR, "models", "qwen2.5-0.5b-instruct-q5_k_m.gguf")
ocr_system = OCREngine()
address_parser = AddressParser(model_path=model_path)
return ocr_system, address_parser
ocr_system, address_parser = load_engines()
st.title("Spatial Address Extractor & Reviewer")
st.sidebar.header("Input Source")
input_type = st.sidebar.radio("Select Image Source:", ("Upload File", "Take Camera Picture"))
uploaded_image = None
if input_type == "Upload File":
uploaded_image = st.sidebar.file_uploader("Choose an address image...", type=["jpg", 
"jpeg", "png"])
else:
uploaded_image = st.sidebar.camera_input("Capture address plate")
if "parsed_data" not in st.session_state:
st.session_state.parsed_data = None
if "current_image_name" not in st.session_state:
st.session_state.current_image_name = None
if uploaded_image is not None:
img_id = uploaded_image.name if hasattr(uploaded_image, 'name') else "camera_capture"
if st.session_state.current_image_name != img_id:
st.session_state.current_image_name = img_id
st.session_state.parsed_data = None
image = Image.open(uploaded_image)
temp_path = os.path.join("data", "temp_runtime_image.jpg")
image.save(temp_path)
if st.session_state.parsed_data is None:
with st.spinner("Executing EasyOCR and Qwen-0.5B Extraction..."):
raw_text = ocr_system.extract_text(temp_path)
if raw_text:
st.session_state.parsed_data = address_parser.parse_ocr_text(raw_text)
else:
st.session_state.parsed_data = {
"pincode": "", "building_or_house": "", 
"street_or_locality": "", "city": "", "state": ""
}
col1, col2 = st.columns([1, 1])
with col1:
st.subheader("Source Image")
st.image(image, use_column_width=True)
with col2:
st.subheader("Extracted JSON Fields (Review & Edit)")
edited_data = {}
edited_data["pincode"] = st.text_input("Pincode / Postal Code", 
st.session_state.parsed_data.get("pincode", ""))
edited_data["building_or_house"] = st.text_input("Building / House / Flat No.", 
st.session_state.parsed_data.get("building_or_house", ""))
edited_data["street_or_locality"] = st.text_input("Street / Locality / Road", 
st.session_state.parsed_data.get("street_or_locality", ""))
edited_data["city"] = st.text_input("City / Town / District", 
st.session_state.parsed_data.get("city", ""))
edited_data["state"] = st.text_input("State", 
st.session_state.parsed_data.get("state", ""))
st.markdown("---")
# Action handler to output final verified data structure
if st.button("Confirm & Proceed"):
st.success("JSON verified successfully!")
st.subheader("Final Edited JSON Output")
st.json(edited_data)
else:
st.info("Please upload an image or capture a photo from the sidebar to begin processing.")
