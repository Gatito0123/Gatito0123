# app.py
import streamlit as st
import numpy as np
import plotly.express as px
import firebase_admin
from firebase_admin import credentials, auth, firestore
import pandas as pd
from fpdf import FPDF
from datetime import datetime

# Configuración Firebase
if not firebase_admin._apps:
    cred = credentials.Certificate("serviceAccountKey.json")
    firebase_admin.initialize_app(cred)

db = firestore.client()

# Componentes Cuánticos
zero_state = np.array([[1], [0]])
H = 1/np.sqrt(2) * np.array([[1, 1], [1, -1]])

def apply_gate(state, gate):
    return np.dot(gate, state)

# Algoritmos
def grover_search():
    state = np.kron(zero_state, zero_state)
    state = apply_gate(np.kron(H, H), state)
    oracle = np.eye(4)
    oracle[-1, -1] = -1
    diffusion = 2 * np.full((4,4), 1/4) - np.eye(4)
    state = apply_gate(oracle, state)
    state = apply_gate(diffusion, state)
    return state

# Interfaz
def main():
    st.set_page_config(page_title="Michael.bo Quantum", layout="wide")
    
    # Diseño personalizado
    st.markdown("""
    <style>
    [data-testid="stAppViewContainer"] {background: url('https://raw.githubusercontent.com/MichaelBo-quantum/simulador/main/bg.jpg');}
    .main-title {color: #fff; text-align: center; font-size: 3.5rem; text-shadow: 2px 2px 4px #000;}
    .stButton>button {background: linear-gradient(45deg, #2196F3, #E91E63); color: white;}
    </style>
    """, unsafe_allow_html=True)
    
    # Autenticación
    if 'user' not in st.session_state:
        st.session_state.user = None
    
    if not st.session_state.user:
        with st.container():
            col1, col2, col3 = st.columns([1,2,1])
            with col2:
                st.markdown('<h1 class="main-title">🔮 Michael.bo Quantum</h1>', unsafe_allow_html=True)
                auth_type = st.radio("", ["Registrarse", "Iniciar Sesión"])
                email = st.text_input("✉️ Correo")
                password = st.text_input("🔑 Contraseña", type="password")
                
                if st.button("Acceder"):
                    try:
                        if auth_type == "Registrarse":
                            user = auth.create_user(email=email, password=password)
                            st.success("¡Registro exitoso! ✅")
                        else:
                            user = auth.get_user_by_email(email)
                            st.session_state.user = email
                            db.collection("users").document(email).set({
                                "last_login": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                                "usage_count": firestore.Increment(1)
                            }, merge=True)
                            st.experimental_rerun()
                    except Exception as e:
                        st.error(f"Error: {str(e)}")
        return

    # Aplicación principal
    st.sidebar.title(f"👨💻 {st.session_state.user}")
    if st.sidebar.button("🚪 Cerrar Sesión"):
        st.session_state.user = None
        st.experimental_rerun()
    
    menu = st.sidebar.selectbox("Menú", ["🏠 Inicio", "⚛️ Simulador", "📊 Exportar"])
    
    if menu == "⚛️ Simulador":
        st.header("⚡ Simulador Cuántico Michael.bo")
        st.write("**Selecciona un algoritmo:**")
        
        if st.button("🚀 Ejecutar Algoritmo de Grover"):
            result = grover_search()
            probs = np.abs(result.flatten())**2
            fig = px.bar(x=["00", "01", "10", "11"], y=probs, 
                        labels={"x": "Estado Cuántico", "y": "Probabilidad"},
                        color=probs, color_continuous_scale="rainbow")
            st.plotly_chart(fig, use_container_width=True)
            
    elif menu == "📊 Exportar":
        st.header("📤 Exportar Resultados")
        st.write("Próximamente: Exportación a PDF/CSV con logo Michael.bo")
        st.image("https://raw.githubusercontent.com/MichaelBo-quantum/simulador/main/export.png", width=300)

if __name__ == "__main__":
    main()
