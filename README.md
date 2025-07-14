import streamlit as st
import calendar
from datetime import datetime, timedelta, date
import psycopg2
from collections import defaultdict

# Banco de dados
DB_URL = "postgresql://postgres.ryhrgskisigwwqswguza:Aliezel512%23@aws-0-sa-east-1.pooler.supabase.com:6543/postgres"

# Estilos CSS
style_css = """
<style>
/* Estilos para tema claro (light mode) - padrão */

.calendar-header {
  font-weight: bold;
  text-align: center;
  padding: 8px 0;
  color: #222;
  background-color: #ddd;
  border-radius: 4px;
  margin-bottom: 4px;
}

.calendar-day {
  background-color: #f9f9f9;
  border-radius: 10px;
  color: #222;
  text-align: center;
  padding: 10px 5px;
  font-size: 14px;
  min-height: 75px;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  margin: 3px;
  transition: background-color 0.3s;
  box-shadow: 0 0 3px rgba(0,0,0,0.1);
}
.day-number {
  font-weight: bold;
  font-size: 18px;
}
.day-value {
  font-weight: bold;
  font-size: 16px;
  margin-top: 6px;
  color: #0077cc;
  font-family: monospace;
}

/* Status claro */
.status-hoje { border: 3px solid #d1b000; font-weight: bold; }
.status-fds { background-color: #a9cce3; }
.status-feriado { background-color: #f5b7b1; color: #5a2a27; font-weight: bold; }
.status-pendente { box-shadow: 0 0 6px 2px #3399ff; }

/* Totais */
.week-total {
  text-align: right;
  font-size: 14px;
  color: #444;
  font-family: monospace;
  margin-bottom: 25px;
  padding-right: 10px;
}

/* Sidebar */
.sidebar-container {
  padding: 10px;
  margin-bottom: 10px;  /* DIMINUI ESPAÇO ENTRE MENU E CALENDÁRIO */
}

/* Container da imagem com margem superior negativa para subir a imagem */
.sidebar-logo-container {
  margin-top: -20px;  /* Ajuste para subir imagem */
  margin-bottom: 12px;
}

/* Estilo do st.radio para tema claro */
div[role="radiogroup"] > label {
  background-image: linear-gradient(90deg, rgba(42,123,155,1) 0%, rgba(87,199,133,1) 50%, rgba(237,221,83,1) 100%);
  padding: 8px 16px;
  border-radius: 20px;
  margin-right: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  border: 1px solid #ccc;
  color: black;
}
div[role="radiogroup"] > label:hover {
  background-image: linear-gradient(90deg, rgba(42,123,155,1) 0%, rgba(87,199,133,1) 50%, rgba(237,221,83,1) 100%);
  border-color: #3399ff;
}
div[role="radiogroup"] > label[data-selected="true"] {
  background-color: #3399ff;
  color: black;
  font-weight: bold;
  border: 1px solid #0077ff;
}

/* Estilos para tema escuro (dark mode) */
@media (prefers-color-scheme: dark) {
  .calendar-header {
    color: white;
    background-color: #333;
  }

  .calendar-day {
    background-color: #2c2c2c;
    color: white;
    box-shadow: none;
  }

  .day-value {
    color: #a0d8f7;
  }

  /* Status escuro */
  .status-hoje { border: 3px solid #ffee00; font-weight: bold; }
  .status-fds { background-color: #001f3f; }
  .status-feriado { background-color: #ff6f91; color: white; font-weight: bold; }
  .status-pendente { box-shadow: 0 0 6px 2px #0077ff; }

  .week-total {
    color: #ccc;
  }

  /* Sidebar */
  .sidebar-title {
    color: #3399ff;
  }

  /* st.radio escuro */
  div[role="radiogroup"] > label {
    background-image: none;
    background-color: #222;
    color: white;
    border: 1px solid #555;
  }
  div[role="radiogroup"] > label:hover {
    background-color: #3399ff;
    border-color: #3399ff;
    color: black;
  }
  div[role="radiogroup"] > label[data-selected="true"] {
    background-color: #3399ff;
    color: black;
    font-weight: bold;
    border: 1px solid #0077ff;
  }
}
</style>
"""

st.set_page_config(layout="wide", page_title="Resumo Semanal de Boletos")
st.markdown(style_css, unsafe_allow_html=True)

# === LAYOUT ===

col1, col2 = st.columns([1, 4])

with col1:
    st.markdown("<div class='sidebar-container'>", unsafe_allow_html=True)
    
    # Coloca a imagem dentro de um container com margem customizada para subir
    st.markdown("<div class='sidebar-logo-container'>", unsafe_allow_html=True)
    st.image("logo/LOGO.JPG", use_container_width=True)
    st.markdown("</div>", unsafe_allow_html=True)

    st.markdown("<div class='sidebar-title'>Ano</div>", unsafe_allow_html=True)
    ano = st.selectbox("", list(range(2025, 2035)), index=0)

    st.markdown("<div class='sidebar-title'>Mês</div>", unsafe_allow_html=True)
    meses_nomes = ["Janeiro", "Fevereiro", "Março", "Abril", "Maio", "Junho",
                   "Julho", "Agosto", "Setembro", "Outubro", "Novembro", "Dezembro"]
    mes_index = st.radio("", options=list(range(1,13)),
                         format_func=lambda x: meses_nomes[x-1], index=datetime.now().month-1)
    st.markdown("</div>", unsafe_allow_html=True)

with col2:
    st.markdown("## 📊 Resumo Semanal de Boletos")  # Título separado da imagem

    filtro = st.text_input("🔍 Filtro por descrição")

    # Funções para feriados, dados e limites seguem iguais
    FERIADOS_FIXOS = {(1,1),(21,4),(1,5),(7,9),(12,10),(2,11),(15,11),(20,11),(25,12),(15,7),(8,9)}

    def calcular_pascoa(ano):
        a = ano % 19
        b = ano // 100
        c = ano % 100
        d = b // 4
        e = b % 4
        f = (b + 8) // 25
        g = (b - f + 1) // 3
        h = (19 * a + b - d - g + 15) % 30
        i = c // 4
        k = c % 4
        l = (32 + 2 * e + 2 * i - h - k) % 7
        m = (a + 11 * h + 22 * l) // 451
        mes = (h + l - 7 * m + 114) // 31
        dia = ((h + l - 7 * m + 114) % 31) + 1
        return date(ano, mes, dia)

    def feriados_moveis(ano):
        pascoa = calcular_pascoa(ano)
        return {
            pascoa,
            pascoa - timedelta(days=47),
            pascoa - timedelta(days=2),
            pascoa + timedelta(days=60),
        }

    def is_feriado(d):
        return (d.day, d.month) in FERIADOS_FIXOS or d in feriados_moveis(d.year)

    def proximo_dia_util(d):
        while d.weekday() >= 5 or is_feriado(d):
            d += timedelta(days=1)
        return d

    @st.cache_data(ttl=300)
    def obter_dados():
        conn = psycopg2.connect(DB_URL)
        cursor = conn.cursor()
        cursor.execute("SELECT descricao, data, valor FROM lancamentos WHERE status='pendente'")
        dados = cursor.fetchall()
        conn.close()
        return dados

    @st.cache_data(ttl=300)
    def obter_limites(ano, mes):
        conn = psycopg2.connect(DB_URL)
        cursor = conn.cursor()
        cursor.execute("SELECT semana, limite FROM limites_semanais WHERE ano=%s AND mes=%s", (ano, mes))
        res = cursor.fetchall()
        conn.close()
        return dict(res)

    def salvar_limite(ano, mes, semana, limite):
        conn = psycopg2.connect(DB_URL)
        cursor = conn.cursor()
        cursor.execute('''
            INSERT INTO limites_semanais (ano, mes, semana, limite)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (ano, mes, semana) DO UPDATE SET limite = EXCLUDED.limite
        ''', (ano, mes, semana, limite))
        conn.commit()
        conn.close()

    dados = obter_dados()
    resumo = defaultdict(float)
    detalhes = defaultdict(list)

    for desc, data_str, valor in dados:
        try:
            d = datetime.strptime(str(data_str), "%Y-%m-%d").date()
        except:
            continue
        if d.month != mes_index or d.year != ano:
            continue
        d_ajustado = proximo_dia_util(d)
        if d_ajustado.month != mes_index:
            continue
        if filtro.lower() not in desc.lower():
            continue
        resumo[d_ajustado] += float(valor)
        detalhes[d_ajustado].append(desc)

    limites = obter_limites(ano, mes_index)
    semanas = calendar.Calendar(firstweekday=6).monthdatescalendar(ano, mes_index)
    dias_semana = ["Domingo", "Segunda", "Terça", "Quarta", "Quinta", "Sexta", "Sábado"]

    # Cabeçalho dos dias da semana alinhado com o calendário
    header_cols = st.columns(7)
    for idx, dia in enumerate(dias_semana):
        with header_cols[idx]:
            st.markdown(f"<div class='calendar-header'>{dia}</div>", unsafe_allow_html=True)

    for i, semana in enumerate(semanas, start=1):
        cols = st.columns(7)
        total_semana = 0
        for idx, dia in enumerate(semana):
            style = "calendar-day"
            valor_dia = resumo.get(dia, 0.0)
            total_semana += valor_dia

            if dia == date.today():
                style += " status-hoje"
            if is_feriado(dia):
                style += " status-feriado"
            elif dia.weekday() >= 5:
                style += " status-fds"
            elif valor_dia > 0:
                style += " status-pendente"

            if dia.month == mes_index:
                texto_html = (
                    f"<div class='day-number'>{dia.day}</div>"
                    f"<div class='day-value'>R$ {valor_dia:,.2f}</div>"
                )
            else:
                texto_html = ""

            with cols[idx]:
                st.markdown(f"<div class='{style}'>{texto_html}</div>", unsafe_allow_html=True)

        limite = limites.get(i, 0.0)
        cor = "#0f0" if total_semana <= limite else "#f33"
        st.markdown(f"<div class='week-total'>💵 Semana {i} — Total: <b style='color:{cor}'>R$ {total_semana:,.2f}</b> | Limite: R$ {limite:,.2f}</div>", unsafe_allow_html=True)

    with st.expander("⚙️ Editar Limites Semanais"):
        with st.form(key="form_limites_totais"):
            limites_novos = {}
            for i in range(1, len(semanas)+1):
                limite_atual = limites.get(i, 0.0)
                limite_novo = st.number_input(f"Novo limite semana {i}", value=float(limite_atual), key=f"limite_{ano}_{mes_index}_{i}")
                limites_novos[i] = limite_novo
            submit = st.form_submit_button("Salvar todos limites")
            if submit:
                for i, val in limites_novos.items():
                    salvar_limite(ano, mes_index, i, val)
                st.success("Limites semanais salvos com sucesso!")
