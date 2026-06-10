import streamlit as st
import serial
import serial.tools.list_ports
import pandas as pd
import time
from collections import defaultdict, deque
from datetime import datetime

# =========================
# CONFIGURAÇÃO DA PÁGINA
# =========================

st.set_page_config(
    page_title="CAN Trace Monitor - ESP32",
    layout="wide"
)

st.title("CAN Trace Monitor - ESP32")
st.caption("Leitura CAN via ESP32 Serial - Mini BUSMASTER em Streamlit")

# =========================
# CONFIGURAÇÕES LATERAIS
# =========================

st.sidebar.header("Configuração")

available_ports = [port.device for port in serial.tools.list_ports.comports()]

default_port = "COM10"

selected_port = st.sidebar.selectbox(
    "Porta Serial",
    options=available_ports if available_ports else [default_port],
    index=available_ports.index(default_port) if default_port in available_ports else 0
)

baud_rate = st.sidebar.selectbox(
    "Baud Rate Serial",
    options=[9600, 19200, 38400, 57600, 115200, 230400, 500000, 921600],
    index=4
)

max_frames = st.sidebar.number_input(
    "Máximo de frames no buffer",
    min_value=100,
    max_value=100000,
    value=5000,
    step=100
)

filter_id = st.sidebar.text_input(
    "Filtro por ID CAN",
    placeholder="Ex: 0x0FC ou FC"
)

auto_scroll = st.sidebar.checkbox("Atualização automática", value=True)

refresh_rate = st.sidebar.slider(
    "Tempo de atualização",
    min_value=0.1,
    max_value=2.0,
    value=0.5,
    step=0.1
)

# =========================
# ESTADO DA SESSÃO
# =========================

if "running" not in st.session_state:
    st.session_state.running = False

if "frames" not in st.session_state:
    st.session_state.frames = deque(maxlen=max_frames)

if "serial_conn" not in st.session_state:
    st.session_state.serial_conn = None

if "start_time" not in st.session_state:
    st.session_state.start_time = None

if "last_timestamp_by_id" not in st.session_state:
    st.session_state.last_timestamp_by_id = {}

if "count_by_id" not in st.session_state:
    st.session_state.count_by_id = defaultdict(int)

# =========================
# FUNÇÕES
# =========================

def parse_can_line(line):
    """
    Formato esperado:
    timestamp_ms;id;dlc;data

    Exemplo:
    123456;0x0FC;8;11 22 33 44 55 66 77 88
    """

    try:
        parts = line.strip().split(";")

        if len(parts) != 4:
            return None

        timestamp_ms = parts[0].strip()
        can_id = parts[1].strip().upper()
        dlc = int(parts[2].strip())
        data = parts[3].strip().upper()

        if not can_id.startswith("0X"):
            can_id = "0x" + can_id

        can_id = can_id.upper().replace("0X", "0x")

        return {
            "PC Time": datetime.now().strftime("%H:%M:%S.%f")[:-3],
            "ESP Time ms": timestamp_ms,
            "ID": can_id,
            "DLC": dlc,
            "DATA": data
        }

    except Exception:
        return None


def connect_serial(port, baud):
    try:
        ser = serial.Serial(
            port=port,
            baudrate=baud,
            timeout=0.05
        )
        return ser
    except Exception as e:
        st.error(f"Erro ao conectar na porta {port}: {e}")
        return None


def disconnect_serial():
    if st.session_state.serial_conn:
        try:
            st.session_state.serial_conn.close()
        except Exception:
            pass

    st.session_state.serial_conn = None
    st.session_state.running = False


def read_serial_frames():
    ser = st.session_state.serial_conn

    if ser is None or not ser.is_open:
        return

    lines_read = 0

    while ser.in_waiting and lines_read < 500:
        try:
            raw_line = ser.readline().decode(errors="ignore").strip()

            if raw_line:
                frame = parse_can_line(raw_line)

                if frame:
                    can_id = frame["ID"]

                    st.session_state.count_by_id[can_id] += 1
                    frame["Count"] = st.session_state.count_by_id[can_id]

                    now = time.time()

                    if can_id in st.session_state.last_timestamp_by_id:
                        delta = now - st.session_state.last_timestamp_by_id[can_id]
                        frame["Hz"] = round(1 / delta, 2) if delta > 0 else 0
                    else:
                        frame["Hz"] = 0

                    st.session_state.last_timestamp_by_id[can_id] = now
                    st.session_state.frames.append(frame)

            lines_read += 1

        except Exception:
            break


def get_dataframe():
    df = pd.DataFrame(list(st.session_state.frames))

    if df.empty:
        return df

    if filter_id:
        filt = filter_id.strip().upper().replace("0X", "")
        df = df[df["ID"].str.upper().str.replace("0X", "").str.contains(filt)]

    return df


def get_statistics(df):
    if df.empty:
        return pd.DataFrame()

    stats = (
        df.groupby("ID")
        .agg(
            Frames=("ID", "count"),
            Last_DLC=("DLC", "last"),
            Last_DATA=("DATA", "last"),
            Avg_Hz=("Hz", "mean"),
            Last_Hz=("Hz", "last")
        )
        .reset_index()
    )

    stats["Avg_Hz"] = stats["Avg_Hz"].round(2)
    stats["Last_Hz"] = stats["Last_Hz"].round(2)

    return stats.sort_values(by="Frames", ascending=False)

# =========================
# BOTÕES PRINCIPAIS
# =========================

col_btn1, col_btn2, col_btn3 = st.columns(3)

with col_btn1:
    if st.button("Conectar / Iniciar", use_container_width=True):
        if st.session_state.serial_conn is None:
            st.session_state.serial_conn = connect_serial(selected_port, baud_rate)

        if st.session_state.serial_conn:
            st.session_state.running = True
            st.session_state.start_time = time.time()
            st.success(f"Conectado em {selected_port} @ {baud_rate}")

with col_btn2:
    if st.button("Parar / Desconectar", use_container_width=True):
        disconnect_serial()
        st.warning("Conexão encerrada")

with col_btn3:
    if st.button("Limpar Trace", use_container_width=True):
        st.session_state.frames.clear()
        st.session_state.count_by_id.clear()
        st.session_state.last_timestamp_by_id.clear()
        st.success("Trace limpo")

# =========================
# LEITURA SERIAL
# =========================

if st.session_state.running:
    read_serial_frames()

df_trace = get_dataframe()
df_stats = get_statistics(df_trace)

# =========================
# STATUS
# =========================

total_frames = len(st.session_state.frames)
active_ids = df_trace["ID"].nunique() if not df_trace.empty else 0

elapsed = 0
if st.session_state.start_time:
    elapsed = time.time() - st.session_state.start_time

frames_per_second = round(total_frames / elapsed, 2) if elapsed > 0 else 0

status_col1, status_col2, status_col3, status_col4 = st.columns(4)

status_col1.metric("Status", "Rodando" if st.session_state.running else "Parado")
status_col2.metric("Frames", total_frames)
status_col3.metric("IDs ativos", active_ids)
status_col4.metric("Frames/s", frames_per_second)

# =========================
# ABAS
# =========================

tab_trace, tab_stats, tab_export = st.tabs(
    ["Trace CAN", "Estatísticas", "Exportação"]
)

with tab_trace:
    st.subheader("Trace em tempo real")

    if df_trace.empty:
        st.info("Nenhum frame recebido ainda.")
    else:
        st.dataframe(
            df_trace.sort_index(ascending=False),
            use_container_width=True,
            height=500
        )

with tab_stats:
    st.subheader("Estatísticas por ID")

    if df_stats.empty:
        st.info("Sem estatísticas disponíveis.")
    else:
        st.dataframe(
            df_stats,
            use_container_width=True,
            height=500
        )

with tab_export:
    st.subheader("Exportar Trace")

    if df_trace.empty:
        st.info("Nenhum dado para exportar.")
    else:
        csv_data = df_trace.to_csv(index=False).encode("utf-8")

        st.download_button(
            label="Baixar CSV",
            data=csv_data,
            file_name=f"can_trace_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv",
            mime="text/csv"
        )

# =========================
# AUTO REFRESH
# =========================

if auto_scroll and st.session_state.running:
    time.sleep(refresh_rate)
    st.rerun()
