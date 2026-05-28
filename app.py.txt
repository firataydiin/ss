# app.py
import streamlit as st
import pandas as pd
import numpy as np

st.set_page_config(page_title="Formation Rehberi", layout="wide")

# -----------------------
# ÖRNEK VERİLER (daha sonra CSV ile değiştirilebilir)
# -----------------------

# Formation karşılıkları – görsellerdeki haritadan örnekler
match_rows = [
    # enemy, you_send, note, verdict
    ("992/442 INF PHX", "794 INF PHX", "Yüksek INF karşısı", "take"),
    ("992/442 INF PHX", "424 CAV PHX", "Cav phx varyantı", "might"),
    ("992/442 INF PHX", "0 5 5 RNG PHX", "Range phx / pull", "pull"),
    ("424 RNG PHAL", "947 CAV PHX", "Cav baskın", "take"),
    ("424 RNG PHAL", "857 RNG PHAL", "Aynı phalanx", "take"),
    ("424 RNG PHAL", "749 RNG PHAL", "Eşdeğer varyant", "might"),
    ("424 RNG PHAL", "208 RANGE WEDGE", "Wedge riskli", "pull"),
    ("794 INF PHX", "424 RNG PHAL", "Range üstünse", "take"),
    ("794 INF PHX", "208 RANGE WEDGE", "Ağır range baskısı", "might"),
    ("208 RANGE WEDGE", "974 RNG PHAL", "Wedge’e karşı range", "take"),
    ("947 CAV PHX", "0 5 5 INF PHAL", "Cav’a karşı inf", "take"),
]
match_df = pd.DataFrame(match_rows, columns=["Enemy Formation", "You Send", "Comment", "Verdict"])

# Renk haritası
VERDICT_COLOR = {"take": "#1db954", "might": "#ffb000", "pull": "#ff4d4f"}

# Bina mana tablosu – görseldeki toplam satırlarını temel alan örnek
build_rows = [
    ("Mana Lode", [145, 250, 430, 735]),
    ("Castle", [1210, 1815, 2725, 4090]),
    ("Lunar", [1245, 2120, 3615, 6130]),
    ("Farm", [30, 45, 75, 135]),
    ("Lumber", [30, 45, 75, 135]),
    ("Quarry", [30, 45, 75, 135]),
    ("Manor", [155, 235, 355, 530]),
    ("Trading Post", [715, 1065, 1615, 2415]),
    ("Barracks", [390, 585, 880, 1325]),
    ("Infirmary", [440, 650, 880, 1325]),
    ("Embassy", [710, 1065, 1595, 2395]),
    ("Workshop", [635, 955, 1415, 2145]),
    ("Castle Wall", [1120, 1680, 2520, 3785]),
    ("Academy", [695, 1040, 1565, 2345]),
]
build_df = pd.DataFrame(build_rows, columns=["Building", "L1-4"])
for i, lvl in enumerate(["LVL1", "LVL2", "LVL3", "LVL4"]):
    build_df[lvl] = build_df["L1-4"].apply(lambda x: x[i])
build_df.drop(columns=["L1-4"], inplace=True)
build_df["Total(1-4)"] = build_df[["LVL1","LVL2","LVL3","LVL4"]].sum(axis=1)

# Champion set tablosu – slot başına puan (örnek)
champ_cols = ["Slot","Helm","Main Hand","Off Hand","Chest","Boots","Accessory"]
champ_data = [
    [1,20,22,16,28,14,38],
    [2,30,33,24,42,21,57],
    [3,35,39,28,49,25,67],
    [4,80,88,64,112,56,152],
    [5,95,105,76,133,67,181],
    [6,110,121,88,154,77,209],
    [7,170,187,136,238,119,323],
    [8,190,209,152,266,133,361],
    [9,210,231,168,294,147,399],
    [10,290,319,232,406,203,589],
    [11,330,341,248,434,217,627],
    [12,410,451,328,574,287,779],
    [13,570,649,472,826,413,1121],
    [14,890,979,712,1264,623,1691],
    [15,0,0,0,0,0,0],  # boş bırakıldı; gerekirse güncellenir
]
champ_df = pd.DataFrame(champ_data, columns=champ_cols)

# -----------------------
# YARDIMCI FONKSİYONLAR
# -----------------------
def verdict_badge(v):
    color = VERDICT_COLOR.get(v, "#999")
    return f":{v}:" if v not in VERDICT_COLOR else f"<span style='background:{color};color:#000;padding:3px 8px;border-radius:6px;font-weight:600'>{v.upper()}</span>"

def suggest_counter(enemy, comp_inf, comp_rng, comp_cav):
    # Basit kural: önce tablo eşleşmesi, yoksa kompozisyona göre öner
    row = match_df[match_df["Enemy Formation"].str.lower()==enemy.lower()]
    if len(row):
        r = row.iloc[0]
        return r["You Send"], r["Comment"], r["Verdict"]
    # Heuristik
    dom = max([("INF",comp_inf),("RNG",comp_rng),("CAV",comp_cav)], key=lambda x:x[1])[0]
    if "CAV" in enemy:
        return "0 5 5 INF PHAL", "Cav’a karşı inf ağırlık iyi çalışır.", "take"
    if "INF" in enemy:
        return "424 RNG PHAL", "İnf’e karşı range baskısı önerilir.", "might" if dom!="RNG" else "take"
    if "RANGE" in enemy or "RNG" in enemy:
        return "947 CAV PHX", "Range’a karşı cav baskın.", "might" if dom!="CAV" else "take"
    return "424 RNG PHAL", "Varsayılan güvenli seçim.", "might"

# -----------------------
# ARAYÜZ
# -----------------------
st.title("Web Tabanlı Formation ve Kaynak Rehberi")

tabs = st.tabs(["Karşılaşma Simülatörü", "Bina Mana Hesaplayıcı", "Ekipman Gelişim", "Veri Düzenleme"])

with tabs[0]:
    colL, colR = st.columns([1,1])
    with colL:
        st.subheader("Düşman Bilgisi")
        enemy = st.selectbox(
            "Düşman Formation",
            sorted(match_df["Enemy Formation"].unique().tolist()+["424 RNG PHAL","794 INF PHX","947 CAV PHX","208 RANGE WEDGE"])
        )
        st.caption("Listede yoksa manuel yazabilirsiniz.")
        enemy = st.text_input("Manuel giriş (boş bırakılırsa üstteki seçilir)", value=enemy)

        st.subheader("Sizin Kompozisyon")
        c1,c2,c3 = st.columns(3)
        inf = c1.number_input("INF %", min_value=0, max_value=100, value=40)
        rng = c2.number_input("RANGE %", min_value=0, max_value=100, value=40)
        cav = c3.number_input("CAV %", min_value=0, max_value=100, value=20)
        if inf + rng + cav != 100:
            st.warning("Toplam %100 olmalı. Değerler öneri amaçlı yine de hesaplanır.")

        st.divider()
        you_send, note, verdict = suggest_counter(enemy, inf, rng, cav)
        st.markdown(f"### Önerilen Karşı Gönderim: **{you_send}**")
        st.markdown(note)
        st.markdown(verdict_badge(verdict), unsafe_allow_html=True)

    with colR:
        st.subheader("Karşılaşma Haritası")
        df_show = match_df.copy()
        df_show["Result"] = df_show["Verdict"].apply(lambda v: f"{v.upper()}")
        st.dataframe(df_show[["Enemy Formation","You Send","Comment","Result"]], use_container_width=True, height=420)

with tabs[1]:
    st.subheader("Bina Mana Hesaplayıcı")
    st.caption("Seçtiğiniz seviyeye kadar toplam mana hesaplanır.")
    target_lvl = st.select_slider("Hedef Seviye", options=[1,2,3,4], value=4)
    level_cols = ["LVL1","LVL2","LVL3","LVL4"][:target_lvl]
    calc_df = build_df.copy()
    calc_df["Toplam"] = calc_df[level_cols].sum(axis=1)
    total_mana = int(calc_df["Toplam"].sum())
    st.metric("Toplam Mana (seçilen seviyeye kadar)", f"{total_mana:,}")
    st.dataframe(calc_df[["Building"]+level_cols+["Toplam"]], use_container_width=True, height=500)

with tabs[2]:
    st.subheader("Champion Set Gelişim")
    st.caption("Slot seviyelerini seçin; toplam puanlar hesaplanır.")
    levels = {}
    g1,g2,g3 = st.columns(3)
    for i,slot in enumerate(["Helm","Main Hand","Off Hand","Chest","Boots","Accessory"]):
        col = [g1,g2,g3,g1,g2,g3][i]
        levels[slot] = col.slider(f"{slot} seviye", 1, 14, 4)
    def get_val(slot, lvl):
        return champ_df.loc[champ_df["Slot"]==lvl, slot].values[0]
    totals = {slot: get_val(slot, levels[slot]) for slot in levels}
    total_score = sum(totals.values())
    st.metric("Toplam Set Puanı", f"{total_score:,}")
    st.write(pd.DataFrame([totals]))

with tabs[3]:
    st.subheader("Veri Güncelle")
    st.caption("Kendi CSV dosyalarınızı yükleyebilirsiniz. Sütun adları örneklerle aynı olmalı.")
    up1 = st.file_uploader("Formation eşleşmeleri (CSV)", type=["csv"], key="u1")
    if up1:
        match_df[:] = pd.read_csv(up1)
        st.success("Formation verisi güncellendi.")
    up2 = st.file_uploader("Bina mana (CSV)", type=["csv"], key="u2")
    if up2:
        build_df[:] = pd.read_csv(up2)
        st.success("Bina verisi güncellendi.")
    up3 = st.file_uploader("Champion set (CSV)", type=["csv"], key="u3")
    if up3:
        champ_df[:] = pd.read_csv(up3)
        st.success("Ekipman verisi güncellendi.")
    st.write("Örnek şemalar:")
    st.code("match.csv -> Enemy Formation,You Send,Comment,Verdict\n424 RNG PHAL,947 CAV PHX,Range’a karşı cav,take", language="text")
    st.code("buildings.csv -> Building,LVL1,LVL2,LVL3,LVL4", language="text")
    st.code("champ.csv -> Slot,Helm,Main Hand,Off Hand,Chest,Boots,Accessory", language="text")
