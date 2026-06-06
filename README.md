import streamlit as st
import pypdf
import requests

# Initialize session state to store profiles dynamically for public use
if "profiles" not in st.session_state:
    st.session_state.profiles = {
        "ABC Company Pvt Ltd (Default)": {
            "text": (
                "Company Name: ABC Company Pvt Ltd.\n"
                "Location: Meerut, Uttar Pradesh, India\n"
                "Core Products: Rubber hoses, Extruded rubber profiles, Silicon tubes, Rubber gaskets, O-rings, Auto rubber parts, Rubber mats, and custom molded rubber components.\n"
                "Capabilities: Injection molding, Compression molding, In-house tool room, compounding laboratory.\n"
                "Certifications: ISO 9001:2015, IATF 16949.\n"
                "Typical Eligibility: Can handle orders requiring standard rubber polymer testing, high-temperature resistance, and bulk industrial manufacturing."
            ),
            "link": ""
        }
    }

st.set_page_config(page_title="Tender Squeezer AI (Gemini 3 Edition)", layout="wide")
st.title("📋 Automated Tender Compliance Squeezer")
st.subheader("Personalized Tender Verification | Powered by Gemini 3 Flash ⚡")

# 🔑 API Key input box direct app ke interface par
st.sidebar.markdown("### 🔐 API Authentication")
user_api_key = st.sidebar.text_input("Enter your Gemini API Key here:", type="password")
st.sidebar.info("💡 Google AI Studio se mila hua poora key copy karke yahan paste karein.")

# 🏢 Profile and Catalogue management system
st.sidebar.markdown("### 🏢 Company Profile & Catalogue")
selected_profile_name = st.sidebar.selectbox("Select Active Profile / Catalogue:", options=list(st.session_state.profiles.keys()))

# Option to add custom profiles/links dynamically
with st.sidebar.expander("➕ Add New Profile / Catalogue Link"):
    new_profile_name = st.text_input("Profile/Company Name (e.g., Global Polymers)")
    new_catalogue_link = st.text_input("Catalogue Link / Reference URL")
    new_profile_text = st.text_area("Profile Description / Product Specifications")
    
    if st.button("Save Profile"):
        if new_profile_name and (new_profile_text or new_catalogue_link):
            st.session_state.profiles[new_profile_name] = {
                "text": new_profile_text,
                "link": new_catalogue_link
            }
            st.success(f"Profile '{new_profile_name}' saved successfully!")
            st.rerun()
        else:
            st.error("Please provide a Profile Name and either a description or a link.")

# Fetch details of the currently selected profile
active_profile = st.session_state.profiles[selected_profile_name]

col1, col2 = st.columns(2)
with col1:
    org_name = st.text_input("Organization Name (e.g., NTPC, Indian Railways)")
    tender_no = st.text_input("Tender Number / Reference ID")
with col2:
    uploaded_files = st.file_uploader("Upload Tender Document(s) (PDF)", type=["pdf"], accept_multiple_files=True)

def extract_text_from_pdf(file):
    try:
        pdf_reader = pypdf.PdfReader(file)
        text = ""
        for page in pdf_reader.pages[:40]:
            extracted = page.extract_text()
            if extracted:
                text += extracted + "\n"
        return " ".join(text.split())
    except Exception as e:
        st.error(f"PDF Error: {e}")
        return ""

if st.button("Start Gemini 3 Deep Analysis 🔥") and uploaded_files:
    if not user_api_key:
        st.error("⚠️ Please enter your Gemini API Key in the left sidebar first!")
    elif not org_name or not tender_no:
        st.warning("Please fill Organization Name and Tender Number first!")
    else:
        # 🔄 Modified language to "Analyzing" as requested
        with st.spinner("Analyzing... Please wait."):
            
            combined_text = ""
            for file in uploaded_files:
                combined_text += extract_text_from_pdf(file) + "\n"
            
            safe_context = combined_text[:40000]
            
            if len(safe_context) < 100:
                st.error("PDFs se text thik se extract nahi ho paya.")
            else:
                # Dynamic construction of profile data for the prompt
                profile_context = f"Company Profile: {selected_profile_name}\n"
                if active_profile["text"]:
                    profile_context += f"Details:\n{active_profile['text']}\n"
                if active_profile["link"]:
                    profile_context += f"Catalogue Reference Link: {active_profile['link']}\n"

                prompt = (
                    f"You are an expert legal and technical tender auditor in India. Analyze the following tender details meticulously.\n\n"
                    f"Organization: {org_name}\n"
                    f"Tender No: {tender_no}\n"
                    f"Our Company Profile:\n{profile_context}\n\n"
                    f"Tender Text Content:\n{safe_context}\n\n"
                    f"Provide a detailed compliance report containing:\n"
                    f"1. TENDER TYPE DETERMINATION\n"
                    f"2. ELIGIBILITY MATRIX\n"
                    f"3. PRODUCT MATCH CRITERIA\n"
                    f"4. CRITICAL RISKS & HIDDEN CLAUSES\n"
                    f"5. FINAL VERDICT (GO/NO-GO with confidence score)"
                )
                
                headers = {'Content-Type': 'application/json'}
                payload = {"contents": [{"parts": [{"text": prompt}]}]}
                
                url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key={user_api_key}"
                
                try:
                    response = requests.post(url, headers=headers, json=payload, timeout=45)
                    res_json = response.json()
                    
                    if response.status_code == 200:
                        output_text = res_json['candidates'][0]['content']['parts'][0]['text']
                        st.success("Gemini 3 Engine Analysis Complete!")
                        st.markdown("### 📊 Comprehensive Compliance Report")
                        st.write(output_text)
                    else:
                        error_msg = res_json.get('error', {}).get('message', 'Unknown Error')
                        st.error(f"API Error ({response.status_code}): {error_msg}")
                        st.info("Tip: Ek baar check karein ki aapki copy ki hui key poori sahi hai ya nahi.")
                except Exception as e:
                    st.error(f"Network Error: {e}")
