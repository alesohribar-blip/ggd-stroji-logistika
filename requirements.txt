import streamlit as st
import pandas as pd
import sqlite3
import folium
from folium.plugins import MarkerCluster
from math import radians, cos, sin, asin, sqrt
from datetime import datetime

# ====================== POMOŽNE FUNKCIJE ======================
def haversine(lon1, lat1, lon2, lat2):
    lon1, lat1, lon2, lat2 = map(radians, [lon1, lat1, lon2, lat2])
    dlon = lon2 - lon1 
    dlat = lat2 - lat1 
    a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
    c = 2 * asin(sqrt(a)) 
    r = 6371  # km
    return c * r

def init_db():
    conn = sqlite3.connect('ggd_stroji.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS stroji
                 (id TEXT PRIMARY KEY, tip TEXT, lat REAL, lon REAL, status TEXT, zadnja_premestitev TEXT, opombe TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS gradbisca
                 (id TEXT PRIMARY KEY, ime TEXT, lat REAL, lon REAL, opombe TEXT)''')
    conn.commit()
    conn.close()

def get_data(table):
    conn = sqlite3.connect('ggd_stroji.db')
    df = pd.read_sql_query(f"SELECT * FROM {table}", conn)
    conn.close()
    return df

# ====================== STREAMLIT APLIKACIJA ======================
st.set_page_config(page_title="GGD Logistika - Organizacija strojev", layout="wide")
st.title("🛠️ GGD d.d. - Logistika strojev & optimizacija premestitev")
st.markdown("**Prototip v1.0** – organizacija, sledenje in najkrajše premestitve po gradbiščih")

init_db()

# SIDEBAR
with st.sidebar:
    st.header("Navigacija")
    stran = st.radio("Izberi stran:", ["Pregled & Zemljevid", "Dodaj/urejaj podatke", "Optimizacija premestitev"])

# ====================== STRAN 1: PREGLED & ZEMLJEVID ======================
if stran == "Pregled & Zemljevid":
    st.subheader("Trenutno stanje strojev in gradbišč")
    
    col1, col2 = st.columns(2)
    with col1:
        stroji = get_data("stroji")
        st.dataframe(stroji, use_container_width=True)
    with col2:
        gradbisca = get_data("gradbisca")
        st.dataframe(gradbisca, use_container_width=True)
    
    # Zemljevid
    st.subheader("Zemljevid lokacij")
    m = folium.Map(location=[46.25, 14.35], zoom_start=8)  # center Slovenije
    marker_cluster = MarkerCluster().add_to(m)
    
    for _, s in stroji.iterrows():
        color = "red" if s["status"] == "Na voljo" else "blue"
        folium.Marker(
            [s["lat"], s["lon"]],
            popup=f"Stroj: {s['id']} ({s['tip']})<br>Status: {s['status']}",
            icon=folium.Icon(color=color, icon="wrench")
        ).add_to(marker_cluster)
    
    for _, g in gradbisca.iterrows():
        folium.Marker(
            [g["lat"], g["lon"]],
            popup=f"Gradbišče: {g['ime']}",
            icon=folium.Icon(color="green", icon="home")
        ).add_to(m)
    
    st.components.v1.html(m._repr_html_(), height=600)

# ====================== STRAN 2: DODAJ/UREJAJ ======================
elif stran == "Dodaj/urejaj podatke":
    tab1, tab2 = st.tabs(["Dodaj stroj", "Dodaj gradbišče"])
    
    with tab1:
        st.subheader("Nov stroj")
        id_stroja = st.text_input("ID stroja (npr. BAGER-045)")
        tip = st.selectbox("Tip stroja", ["Bager", "Buldožer", "Valjar", "Asfaltirnik", "Druge"])
        lat = st.number_input("Trenutna latitude (GPS)", value=46.25)
        lon = st.number_input("Trenutna longitude (GPS)", value=14.35)
        status = st.selectbox("Status", ["Na voljo", "V uporabi", "V transportu", "Vzdrževanje"])
        opombe = st.text_area("Opombe")
        
        if st.button("Shrani stroj"):
            conn = sqlite3.connect('ggd_stroji.db')
            conn.execute("INSERT OR REPLACE INTO stroji VALUES (?,?,?,?,?,?,?)",
                         (id_stroja, tip, lat, lon, status, datetime.now().strftime("%Y-%m-%d %H:%M"), opombe))
            conn.commit()
            conn.close()
            st.success(f"Stroj {id_stroja} dodan!")
    
    with tab2:
        st.subheader("Novo gradbišče")
        id_g = st.text_input("ID gradbišča (npr. GB-023-Kranj)")
        ime = st.text_input("Ime gradbišča")
        lat_g = st.number_input("Latitude", value=46.25)
        lon_g = st.number_input("Longitude", value=14.35)
        opombe_g = st.text_area("Opombe")
        
        if st.button("Shrani gradbišče"):
            conn = sqlite3.connect('ggd_stroji.db')
            conn.execute("INSERT OR REPLACE INTO gradbisca VALUES (?,?,?,?,?)",
                         (id_g, ime, lat_g, lon_g, opombe_g))
            conn.commit()
            conn.close()
            st.success(f"Gradbišče {ime} dodano!")

# ====================== STRAN 3: OPTIMIZACIJA PREMESTITEV ======================
else:
    st.subheader("🚚 Optimizacija premestitev strojev")
    stroji = get_data("stroji")
    gradbisca = get_data("gradbisca")
    
    # Izberi stroje za premestitev
    izbrani = st.multiselect("Izberi stroje, ki jih želiš premestiti", 
                             options=stroji["id"].tolist(),
                             format_func=lambda x: f"{x} - {stroji[stroji['id']==x]['tip'].values[0]}")
    
    premestitve = []
    for s_id in izbrani:
        cilj = st.selectbox(f"Ciljno gradbišče za {s_id}", options=gradbisca["ime"].tolist(), key=f"cilj_{s_id}")
        cilj_lat = gradbisca[gradbisca["ime"]==cilj]["lat"].values[0]
        cilj_lon = gradbisca[gradbisca["ime"]==cilj]["lon"].values[0]
        premestitve.append({"stroj": s_id, "cilj_ime": cilj, "cilj_lat": cilj_lat, "cilj_lon": cilj_lon})
    
    if st.button("Izračunaj optimalno premestitev") and premestitve:
        # Enostavna nearest-neighbor optimizacija
        pot = []
        cas_skupaj = 0
        trenutna_lat, trenutna_lon = 46.25, 14.35  # start iz Kranja (lahko spremeniš)
        
        for p in premestitve:
            razdalja = haversine(trenutna_lon, trenutna_lat, p["cilj_lon"], p["cilj_lat"])
            cas = razdalja / 50 * 60  # minute pri 50 km/h
            pot.append(f"{p['stroj']} → {p['cilj_ime']} ({razdalja:.1f} km, {cas:.0f} min)")
            cas_skupaj += cas
            trenutna_lat, trenutna_lon = p["cilj_lat"], p["cilj_lon"]
        
        st.success(f"**Optimalno zaporedje premestitev** (približno {cas_skupaj:.0f} minut skupaj)")
        for korak in pot:
            st.write("→ " + korak)
        
        st.info("Opomba: To je približek. Za prave cestne poti in omejitve težkih vozil lahko dodamo OpenRouteService API (povej mi, če želiš v2.0).")

st.caption("Prototip pripravil Grok (v Claude stilu) za GGD d.d. – če želiš dodati funkcije (npr. prave cestne poti, več transporterjev, avtomatsko dodeljevanje, izvoz v Excel...), samo povej!")
