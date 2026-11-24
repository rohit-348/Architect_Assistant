import io
import sqlite3
import urllib.parse
from datetime import date
import pandas as pd
import streamlit as st

# ==========================================
# PAGE CONFIGURATION & STYLING
# ==========================================
st.set_page_config(
    page_title="Architect Assistant | Studio & Site Management",
    page_icon="🏛️",
    layout="wide",
    initial_sidebar_state="expanded",
)

st.markdown("""
<style>
    .stTabs [data-baseweb="tab-list"] {
        gap: 8px;
    }
    .stTabs [data-baseweb="tab"] {
        padding: 10px 18px;
        border-radius: 6px;
    }
</style>
""", unsafe_allow_html=True)

DB_FILE = "architect_assistant.db"


# ==========================================
# DATABASE INITIALIZATION & HELPERS
# ==========================================
def get_conn():
    return sqlite3.connect(DB_FILE, check_same_thread=False)


def init_db():
    conn = get_conn()
    c = conn.cursor()

    # 1. Projects Table
    c.execute("""
        CREATE TABLE IF NOT EXISTS projects (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            project_name TEXT UNIQUE NOT NULL,
            client_name TEXT,
            site_location TEXT,
            typology TEXT,
            total_budget REAL DEFAULT 0.0,
            architect_fee REAL DEFAULT 0.0,
            status TEXT DEFAULT 'In Progress',
            start_date TEXT,
            notes TEXT
        )
    """)

    # 2. Work Progress & Milestones Table
    c.execute("""
        CREATE TABLE IF NOT EXISTS milestones (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            project_id INTEGER,
            stage_name TEXT,
            weightage_pct REAL DEFAULT 10.0,
            completion_pct REAL DEFAULT 0.0,
            status TEXT DEFAULT 'Pending',
            remarks TEXT,
            updated_on TEXT,
            FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
        )
    """)

    # 3. Materials & BOQ Ledger
    c.execute("""
        CREATE TABLE IF NOT EXISTS materials (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            project_id INTEGER,
            entry_date TEXT,
            category TEXT,
            material_name TEXT,
            unit TEXT,
            qty_required REAL DEFAULT 0.0,
            qty_procured REAL DEFAULT 0.0,
            unit_rate REAL DEFAULT 0.0,
            total_cost REAL DEFAULT 0.0,
            supplier_name TEXT,
            payment_status TEXT DEFAULT 'Unpaid',
            FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
        )
    """)

    # 4. Financial Ledger (Client Receipts & Vendor/Helper/Contractor Payouts)
    c.execute("""
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            project_id INTEGER,
            txn_date TEXT,
            txn_type TEXT,          -- 'Client Receipt' or 'Expense / Payout'
            party_role TEXT,        -- 'Client', 'Contractor', 'Helper / Mason', 'Shopkeeper / Vendor', etc.
            party_name TEXT,
            description TEXT,
            payment_mode TEXT,      -- 'UPI', 'NEFT/RTGS', 'Cheque', 'Cash'
            amount REAL DEFAULT 0.0,
            FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
        )
    """)

    # 5. Site Snags, Drawings & Action Checklist
    c.execute("""
        CREATE TABLE IF NOT EXISTS site_tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            project_id INTEGER,
            created_date TEXT,
            task_type TEXT,         -- 'Drawing Issue', 'Site Inspection / Snag', 'Client Approval', 'Material Order'
            task_detail TEXT,
            assigned_to TEXT,
            priority TEXT,
            is_done INTEGER DEFAULT 0,
            FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE CASCADE
        )
    """)

    conn.commit()
    conn.close()


def seed_sample_data():
    """Populates a realistic sample architectural project if database is empty."""
    conn = get_conn()
    c = conn.cursor()
    c.execute("SELECT COUNT(*) FROM projects")
    if c.fetchone()[0] == 0:
        c.execute("""
            INSERT INTO projects (project_name, client_name, site_location, typology, total_budget, architect_fee, status, start_date, notes)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            "Courtyard Residence - Wardha",
            "Mr. Anand Kulkarni",
            "Civil Lines, Wardha",
            "Residential Bungalow / Villa",
            4500000.0,
            450000.0,
            "In Progress",
            str(date.today()),
            "Exposed brick facade, passive solar courtyard, rainwater harvesting.",
        ))
        pid = c.lastrowid

        sample_milestones = [
            (pid, "1. Concept Design & 3D Visualization", 10, 100, "Completed", "Approved by client", str(date.today())),
            (pid, "2. Municipal Sanction & Working Drawings", 15, 100, "Completed", "Structural drawings issued", str(date.today())),
            (pid, "3. Excavation & Plinth Foundation", 20, 85, "In Progress", "Plinth beam curing underway", str(date.today())),
            (pid, "4. RCC Superstructure & Brick Masonry", 25, 20, "In Progress", "Ground floor columns started", str(date.today())),
            (pid, "5. Electrical, Plumbing & Flooring", 15, 0, "Pending", "Material selection next month", str(date.today())),
            (pid, "6. Interior Finishes & Handover", 15, 0, "Pending", "-", str(date.today())),
        ]
        c.executemany("""
            INSERT INTO milestones (project_id, stage_name, weightage_pct, completion_pct, status, remarks, updated_on)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, sample_milestones)

        sample_materials = [
            (pid, str(date.today()), "Cement & Concrete", "OPC 53 Grade Cement (UltraTech)", "Bags", 400, 250, 395.0, 98750.0, "Shree Balaji Traders", "Paid"),
            (pid, str(date.today()), "Steel & Reinforcement", "TMT Steel Bars Fe550D (12mm & 16mm)", "Kg", 5000, 3000, 66.0, 198000.0, "Vidarbha Steels", "Paid"),
            (pid, str(date.today()), "Masonry & Stone", "Wire-Cut Red Clay Bricks", "Nos", 15000, 8000, 11.5, 92000.0, "Wardha Kiln Works", "Partial"),
            (pid, str(date.today()), "Plumbing & Sanitary", "Astral CPVC Pipes & Fittings", "Lot", 1, 0, 45000.0, 0.0, "Pending", "Unpaid"),
        ]
        c.executemany("""
            INSERT INTO materials (project_id, entry_date, category, material_name, unit, qty_required, qty_procured, unit_rate, total_cost, supplier_name, payment_status)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, sample_materials)

        sample_txns = [
            (pid, str(date.today()), "Client Receipt", "Client", "Mr. Anand Kulkarni", "Mobilization Advance + 1st Stage", "NEFT/RTGS", 800000.0),
            (pid, str(date.today()), "Expense / Payout", "Contractor", "Rameshwar Civil Works", "Foundation & Plinth Labour Bill #1", "NEFT/RTGS", 185000.0),
            (pid, str(date.today()), "Expense / Payout", "Shopkeeper / Vendor", "Shree Balaji Traders", "Cement & Aggregates Payment", "UPI", 98750.0),
            (pid, str(date.today()), "Expense / Payout", "Helper / Mason", "Site Helpers Weekly Wages", "Curing & Site Clearing Labour", "Cash", 14500.0),
        ]
        c.executemany("""
            INSERT INTO transactions (project_id, txn_date, txn_type, party_role, party_name, description, payment_mode, amount)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        """, sample_txns)

        sample_tasks = [
            (pid, str(date.today()), "Drawing Issue", "Issue Ground Floor Electrical Conduit Layout to contractor", "Rameshwar Civil Works", "High", 0),
            (pid, str(date.today()), "Site Inspection / Snag", "Check plinth level waterproofing coat before backfilling", "Site Engineer / Architect", "High", 0),
            (pid, str(date.today()), "Client Approval", "Finalize courtyard Kota stone vs. Terracotta tile sample", "Mr. Anand Kulkarni", "Medium", 0),
        ]
        c.executemany("""
            INSERT INTO site_tasks (project_id, created_date, task_type, task_detail, assigned_to, priority, is_done)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, sample_tasks)

        conn.commit()
    conn.close()


init_db()
seed_sample_data()


# ==========================================
# EXCEL EXPORT & IMPORT (BACKUP & SYNC)
# ==========================================
def export_full_database_excel():
    conn = get_conn()
    buf = io.BytesIO()
    with pd.ExcelWriter(buf, engine="openpyxl") as writer:
        for table in ["projects", "milestones", "materials", "transactions", "site_tasks"]:
            df = pd.read_sql_query(f"SELECT * FROM {table}", conn)
            df.to_excel(writer, index=False, sheet_name=table)
    conn.close()
    buf.seek(0)
    return buf


def import_full_database_excel(uploaded_file):
    conn = get_conn()
    xl = pd.ExcelFile(uploaded_file)
    for table in ["projects", "milestones", "materials", "transactions", "site_tasks"]:
        if table in xl.sheet_names:
            df = xl.parse(table)
            df.to_sql(table, conn, if_exists="replace", index=False)
    conn.close()


def format_inr(amount):
    """Formats numbers in clean Indian currency style (₹)."""
    if pd.isna(amount):
        return "₹0"
    return f"₹{amount:,.0f}"


# ==========================================
# LIVE MATERIAL RATES & MARKETPLACE CATALOG
# ==========================================
LIVE_MARKET_BENCHMARKS = [
    {"Category": "Cement", "Item": "OPC 53 Grade Cement (50 kg Bag)", "Unit": "Bag", "Indicative Market Rate (₹)": "₹380 – ₹430", "Brands": "UltraTech, ACC, Ambuja", "Amazon Query": "cement 50kg bag", "Flipkart Query": "construction cement", "IndiaMART Query": "OPC 53 Grade Cement"},
    {"Category": "Steel", "Item": "TMT Steel Rebar Fe-550D (8mm–20mm)", "Unit": "Kg", "Indicative Market Rate (₹)": "₹62 – ₹72", "Brands": "Tata Tiscon, JSW Neosteel, SAIL", "Amazon Query": "steel binding wire construction", "Flipkart Query": "construction tools", "IndiaMART Query": "TMT Steel Bars Fe550D"},
    {"Category": "Masonry", "Item": "AAC Blocks (600x200x200 mm)", "Unit": "Block", "Indicative Market Rate (₹)": "₹55 – ₹75", "Brands": "Birla Aerocon, Siporex, Magicrete", "Amazon Query": "AAC block adhesive", "Flipkart Query": "block adhesive", "IndiaMART Query": "AAC Blocks"},
    {"Category": "Masonry", "Item": "First-Class Red Clay Bricks", "Unit": "Brick", "Indicative Market Rate (₹)": "₹9 – ₹14", "Brands": "Local Kiln / Wire-Cut", "Amazon Query": "bricklaying trowel spirit level", "Flipkart Query": "masonry tools", "IndiaMART Query": "Wire Cut Red Bricks"},
    {"Category": "Sand & Aggregate", "Item": "Crushed River / M-Sand", "Unit": "Ton", "Indicative Market Rate (₹)": "₹1,100 – ₹1,600", "Brands": "Local Quarry", "Amazon Query": "sand sieve mesh", "Flipkart Query": "construction sieve", "IndiaMART Query": "M Sand Crushed Sand"},
    {"Category": "Flooring", "Item": "GVT Vitrified Tiles (600x1200 mm)", "Unit": "Sq.Ft", "Indicative Market Rate (₹)": "₹55 – ₹135", "Brands": "Kajaria, Somany, Johnson", "Amazon Query": "tile spacers leveling system", "Flipkart Query": "tile cutter machine", "IndiaMART Query": "GVT Vitrified Tiles 600x1200"},
    {"Category": "Paint & Finishes", "Item": "Interior Luxury Emulsion (20L)", "Unit": "Bucket (20L)", "Indicative Market Rate (₹)": "₹4,800 – ₹7,200", "Brands": "Asian Paints Royale, Berger Silk", "Amazon Query": "Asian Paints Royale 20 litre", "Flipkart Query": "Asian Paints interior emulsion", "IndiaMART Query": "Asian Paints Royale 20L"},
    {"Category": "Paint & Finishes", "Item": "Exterior Weatherproof Emulsion (20L)", "Unit": "Bucket (20L)", "Indicative Market Rate (₹)": "₹4,200 – ₹6,500", "Brands": "Asian Paints Apex Ultima, Dulux Weathershield", "Amazon Query": "Asian Paints Apex Ultima 20L", "Flipkart Query": "exterior wall paint 20L", "IndiaMART Query": "Asian Paints Apex Ultima"},
    {"Category": "Electrical", "Item": "FR PVC Insulated Copper Wire (1.5 sq.mm, 90m)", "Unit": "Coil (90m)", "Indicative Market Rate (₹)": "₹1,450 – ₹1,850", "Brands": "Polycab, Havells, Finolex", "Amazon Query": "Polycab 1.5 sq mm wire 90m", "Flipkart Query": "Havells 1.5 sq mm electrical wire", "IndiaMART Query": "Polycab 1.5 sq mm FR wire"},
    {"Category": "Electrical", "Item": "Modular Switches & Sockets (16A/6A)", "Unit": "Piece", "Indicative Market Rate (₹)": "₹95 – ₹280", "Brands": "Schneider Livia, Legrand Myrius, Havells", "Amazon Query": "Legrand Myrius modular switch", "Flipkart Query": "Schneider modular switches", "IndiaMART Query": "Legrand modular switches"},
    {"Category": "Plumbing", "Item": "CPVC SDR-11 Pipe (1 Inch, 3m)", "Unit": "Length (3m)", "Indicative Market Rate (₹)": "₹320 – ₹440", "Brands": "Astral, Ashirvad, Supreme", "Amazon Query": "Astral CPVC pipe fittings", "Flipkart Query": "CPVC pipe cutter", "IndiaMART Query": "Astral CPVC Pipe 1 inch"},
    {"Category": "Sanitaryware", "Item": "Wall-Hung Rimless WC + Concealed Cistern", "Unit": "Set", "Indicative Market Rate (₹)": "₹14,500 – ₹28,000", "Brands": "Jaquar, Kohler, Hindware", "Amazon Query": "Jaquar wall hung commode", "Flipkart Query": "Hindware wall hung toilet", "IndiaMART Query": "Jaquar wall hung WC"},
    {"Category": "Hardware & Doors", "Item": "304 Grade SS Mortise Door Lock Handle Set", "Unit": "Set", "Indicative Market Rate (₹)": "₹1,650 – ₹3,500", "Brands": "Godrej, Dorset, Yale", "Amazon Query": "Godrej mortise door lock handle set", "Flipkart Query": "Dorset mortise lock", "IndiaMART Query": "Godrej mortise lock set"},
    {"Category": "Lighting", "Item": "Architectural COB Recessed Spotlight (12W Warm White)", "Unit": "Piece", "Indicative Market Rate (₹)": "₹550 – ₹1,250", "Brands": "Philips, Wipro, Hybec", "Amazon Query": "Philips 12W COB recessed spotlight warm white", "Flipkart Query": "12W COB ceiling spotlight", "IndiaMART Query": "12W COB architectural spotlight"},
]


# ==========================================
# SIDEBAR: NAVIGATION & DATA BACKUP
# ==========================================
with st.sidebar:
    st.title("🏛️ Architect Assistant")
    st.caption("Your Digital Studio, Site Ledger & Material Companion")

    conn = get_conn()
    projects_df = pd.read_sql_query("SELECT * FROM projects ORDER BY id DESC", conn)
    conn.close()

    st.divider()
    nav_mode = st.radio(
        "Workspace",
        [
            "📊 Studio Overview (All Projects)",
            "🏗️ Project Ledger, BOQ & Site",
            "🛒 Live Material Rates & Store Lookup",
            "➕ Create / Manage Projects",
        ],
    )

    selected_project_id = None
    if not projects_df.empty and nav_mode == "🏗️ Project Ledger, BOQ & Site":
        st.divider()
        proj_names = projects_df["project_name"].tolist()
        chosen_name = st.selectbox("Select Active Site / Project", proj_names)
        selected_project_id = int(projects_df[projects_df["project_name"] == chosen_name].iloc[0]["id"])

    st.divider()
    st.subheader("💾 Master Excel Backup")
    st.caption("Download all projects, khatas, BOQs & site checklists in one Excel file.")
    excel_backup = export_full_database_excel()
    st.download_button(
        label="⬇️ Download Studio Backup (.xlsx)",
        data=excel_backup,
        file_name=f"Architect_Assistant_Backup_{date.today()}.xlsx",
        mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        use_container_width=True,
    )

    with st.expander("Restore from Excel Backup"):
        up_backup = st.file_uploader("Upload Architect Assistant .xlsx", type=["xlsx"], key="restore_xl")
        if up_backup is not None:
            if st.button("⚠️ Restore & Overwrite Data", use_container_width=True):
                import_full_database_excel(up_backup)
                st.success("Studio data restored successfully!")
                st.rerun()


# ==========================================
# VIEW 1: STUDIO OVERVIEW (ALL PROJECTS)
# ==========================================
if nav_mode == "📊 Studio Overview (All Projects)":
    st.title("🏛️ Architect Assistant — Studio Overview")
    st.markdown("Consolidated financial health, site progress, and pending tasks across all your projects.")

    conn = get_conn()
    all_proj = pd.read_sql_query("SELECT * FROM projects", conn)
    all_txns = pd.read_sql_query("SELECT * FROM transactions", conn)
    all_mats = pd.read_sql_query("SELECT * FROM materials", conn)
    all_miles = pd.read_sql_query("SELECT * FROM milestones", conn)
    all_tasks = pd.read_sql_query("SELECT * FROM site_tasks", conn)
    conn.close()

    if all_proj.empty:
        st.info("Welcome to **Architect Assistant**! Go to **Create / Manage Projects** in the sidebar to add your first site.")
    else:
        total_received = all_txns[all_txns["txn_type"] == "Client Receipt"]["amount"].sum()
        total_paid_out = all_txns[all_txns["txn_type"] == "Expense / Payout"]["amount"].sum()
        net_cash_hand = total_received - total_paid_out
        unpaid_materials = all_mats[all_mats["payment_status"] != "Paid"]["total_cost"].sum()
        open_tasks_count = len(all_tasks[all_tasks["is_done"] == 0]) if not all_tasks.empty else 0

        k1, k2, k3, k4, k5 = st.columns(5)
        k1.metric("Active / Total Projects", f"{len(all_proj[all_proj['status']=='In Progress'])} / {len(all_proj)}")
        k2.metric("Taken from Clients", format_inr(total_received))
        k3.metric("Paid to Contractors/Helpers/Shops", format_inr(total_paid_out))
        k4.metric("Net Cash in Hand", format_inr(net_cash_hand), delta=f"Unpaid Mat. Bills: {format_inr(unpaid_materials)}", delta_color="inverse")
        k5.metric("Open Site / Drawing Tasks", f"{open_tasks_count} Pending")

        st.divider()
        st.subheader("📁 Project-Wise Status & Accounts")

        summary_rows = []
        for _, p in all_proj.iterrows():
            pid = p["id"]
            p_txns = all_txns[all_txns["project_id"] == pid]
            p_miles = all_miles[all_miles["project_id"] == pid]

            rec = p_txns[p_txns["txn_type"] == "Client Receipt"]["amount"].sum()
            spent = p_txns[p_txns["txn_type"] == "Expense / Payout"]["amount"].sum()

            if not p_miles.empty and p_miles["weightage_pct"].sum() > 0:
                prog = (p_miles["weightage_pct"] * p_miles["completion_pct"]).sum() / p_miles["weightage_pct"].sum()
            else:
                prog = 0.0

            summary_rows.append({
                "Project Name": p["project_name"],
                "Client": p["client_name"],
                "Location": p["site_location"],
                "Status": p["status"],
                "Work Done (%)": round(prog, 1),
                "Est. Budget (₹)": p["total_budget"],
                "Client Received (₹)": rec,
                "Total Spent (₹)": spent,
                "Project Balance (₹)": rec - spent,
            })

        sum_df = pd.DataFrame(summary_rows)
        st.dataframe(
            sum_df,
            use_container_width=True,
            hide_index=True,
            column_config={
                "Work Done (%)": st.column_config.ProgressColumn("Work Done (%)", min_value=0, max_value=100, format="%.1f%%"),
                "Est. Budget (₹)": st.column_config.NumberColumn(format="₹%,.0f"),
                "Client Received (₹)": st.column_config.NumberColumn(format="₹%,.0f"),
                "Total Spent (₹)": st.column_config.NumberColumn(format="₹%,.0f"),
                "Project Balance (₹)": st.column_config.NumberColumn(format="₹%,.0f"),
            },
        )

        if not all_txns.empty:
            c_left, c_right = st.columns(2)
            with c_left:
                st.subheader("💸 Studio Payouts by Role")
                payouts = all_txns[all_txns["txn_type"] == "Expense / Payout"]
                if not payouts.empty:
                    role_grp = payouts.groupby("party_role")["amount"].sum().reset_index()
                    st.bar_chart(role_grp.set_index("party_role"))
            with c_right:
                st.subheader("🧾 Latest Transactions Across Sites")
                recent = all_txns.merge(all_proj[["id", "project_name"]], left_on="project_id", right_on="id", how="left")
                st.dataframe(
                    recent[["txn_date", "project_name", "txn_type", "party_role", "party_name", "amount"]].head(8),
                    use_container_width=True,
                    hide_index=True,
                )


# ==========================================
# VIEW 2: PROJECT LEDGER, BOQ & SITE
# ==========================================
elif nav_mode == "🏗️ Project Ledger, BOQ & Site":
    if selected_project_id is None:
        st.warning("Please create a project first under **Create / Manage Projects**.")
    else:
        conn = get_conn()
        proj = pd.read_sql_query("SELECT * FROM projects WHERE id = ?", conn, params=(selected_project_id,)).iloc[0]
        miles_df = pd.read_sql_query("SELECT * FROM milestones WHERE project_id = ? ORDER BY id", conn, params=(selected_project_id,))
        mats_df = pd.read_sql_query("SELECT * FROM materials WHERE project_id = ? ORDER BY id DESC", conn, params=(selected_project_id,))
        txns_df = pd.read_sql_query("SELECT * FROM transactions WHERE project_id = ? ORDER BY txn_date DESC, id DESC", conn, params=(selected_project_id,))
        tasks_df = pd.read_sql_query("SELECT * FROM site_tasks WHERE project_id = ? ORDER BY is_done ASC, id DESC", conn, params=(selected_project_id,))
        conn.close()

        st.header(f"🏗️ {proj['project_name']}")
        st.caption(f"👤 Client: **{proj['client_name']}**  |  📍 Site: **{proj['site_location']}**  |  🏛️ Typology: **{proj['typology']}**  |  📌 Status: **{proj['status']}**")

        client_in = txns_df[txns_df["txn_type"] == "Client Receipt"]["amount"].sum()
        total_out = txns_df[txns_df["txn_type"] == "Expense / Payout"]["amount"].sum()
        balance = client_in - total_out
        mat_committed = mats_df["total_cost"].sum()
        mat_pending = mats_df[mats_df["payment_status"] != "Paid"]["total_cost"].sum()

        if not miles_df.empty and miles_df["weightage_pct"].sum() > 0:
            overall_progress = (miles_df["weightage_pct"] * miles_df["completion_pct"]).sum() / miles_df["weightage_pct"].sum()
        else:
            overall_progress = 0.0

        m1, m2, m3, m4, m5 = st.columns(5)
        m1.metric("Overall Work Done", f"{overall_progress:.1f}%")
        m2.metric("Taken from Client", format_inr(client_in), f"Est. Budget: {format_inr(proj['total_budget'])}")
        m3.metric("Total Paid Out", format_inr(total_out))
        m4.metric("Project Cash Balance", format_inr(balance))
        m5.metric("Pending Material Bills", format_inr(mat_pending), f"Total BOQ: {format_inr(mat_committed)}", delta_color="inverse")

        st.progress(min(int(overall_progress), 100))

        tab_money, tab_mat, tab_work, tab_party, tab_tasks = st.tabs([
            "💰 Client & Payout Ledger",
            "🧱 Materials & BOQ",
            "📐 Work Progress",
            "👷 Contractor / Helper / Shop Khata",
            "📋 Drawings & Site Checklist",
        ])

        # ------------------------------------------
        # TAB 1: MONEY LEDGER (IN & OUT)
        # ------------------------------------------
        with tab_money:
            st.subheader("Log Payment Received or Given")
            with st.form("add_txn_form", clear_on_submit=True):
                c1, c2, c3, c4 = st.columns([1.2, 1.3, 1.3, 1.5])
                with c1:
                    t_date = st.date_input("Date", value=date.today())
                with c2:
                    t_type = st.selectbox("Transaction Type", ["Expense / Payout", "Client Receipt"])
                with c3:
                    t_role = st.selectbox(
                        "Person / Role",
                        ["Contractor", "Helper / Mason", "Shopkeeper / Vendor", "Client", "Structural / MEP Consultant", "Architect Fee Withdrawal", "Other"],
                    )
                with c4:
                    t_party = st.text_input("Person / Shop Name *", placeholder="e.g., Rameshwar Contractors / Balaji Hardware")

                c5, c6, c7 = st.columns([2.5, 1, 1.2])
                with c5:
                    t_desc = st.text_input("Work / Bill Description", placeholder="e.g., Plinth concrete labour advance / 100 bags cement")
                with c6:
                    t_mode = st.selectbox("Payment Mode", ["UPI", "NEFT/RTGS", "Cash", "Cheque"])
                with c7:
                    t_amt = st.number_input("Amount (₹) *", min_value=0.0, step=500.0)

                if st.form_submit_button("💾 Save Transaction", type="primary"):
                    if not t_party.strip() or t_amt <= 0:
                        st.error("Please enter a valid Person/Shop Name and Amount greater than 0.")
                    else:
                        conn = get_conn()
                        conn.execute("""
                            INSERT INTO transactions (project_id, txn_date, txn_type, party_role, party_name, description, payment_mode, amount)
                            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                        """, (selected_project_id, str(t_date), t_type, t_role, t_party.strip(), t_desc.strip(), t_mode, float(t_amt)))
                        conn.commit()
                        conn.close()
                        st.success("Transaction logged!")
                        st.rerun()

            st.divider()
            st.subheader("Transaction History")
            filter_type = st.multiselect(
                "Filter by Type",
                ["Client Receipt", "Expense / Payout"],
                default=["Client Receipt", "Expense / Payout"],
            )
            filtered_txns = txns_df[txns_df["txn_type"].isin(filter_type)]
            st.dataframe(
                filtered_txns[["id", "txn_date", "txn_type", "party_role", "party_name", "description", "payment_mode", "amount"]],
                use_container_width=True,
                hide_index=True,
                column_config={
                    "amount": st.column_config.NumberColumn("Amount (₹)", format="₹%,.0f"),
                },
            )

            with st.expander("🗑️ Delete an Incorrect Entry"):
                del_tid = st.number_input("Enter Transaction ID to delete", min_value=0, step=1)
                if st.button("Delete Transaction"):
                    conn = get_conn()
                    conn.execute("DELETE FROM transactions WHERE id = ? AND project_id = ?", (int(del_tid), selected_project_id))
                    conn.commit()
                    conn.close()
                    st.rerun()

        # ------------------------------------------
        # TAB 2: MATERIALS & BOQ TRACKER
        # ------------------------------------------
        with tab_mat:
            st.subheader("Add Required or Purchased Material")
            with st.form("add_mat_form", clear_on_submit=True):
                mc1, mc2, mc3, mc4 = st.columns([1.3, 2, 1, 1.5])
                with mc1:
                    m_cat = st.selectbox(
                        "Category",
                        ["Cement & Concrete", "Steel & Reinforcement", "Masonry & Stone", "Sand & Aggregate", "Flooring & Tiles", "Plumbing & Sanitary", "Electrical & Lighting", "Paint & Woodwork", "Hardware & Misc"],
                    )
                with mc2:
                    m_name = st.text_input("Material Specification *", placeholder="e.g., UltraTech OPC 53 Cement / 12mm TMT")
                with mc3:
                    m_unit = st.selectbox("Unit", ["Bags", "Kg", "Tons", "Nos", "Sq.Ft", "R.Ft", "Brass", "Litres", "Boxes", "Lot"])
                with mc4:
                    m_supp = st.text_input("Shopkeeper / Supplier", placeholder="e.g., Shree Balaji Traders")

                mc5, mc6, mc7, mc8 = st.columns([1, 1, 1, 1.2])
                with mc5:
                    m_req = st.number_input("Qty Required", min_value=0.0, step=1.0)
                with mc6:
                    m_proc = st.number_input("Qty Ordered/Delivered", min_value=0.0, step=1.0)
                with mc7:
                    m_rate = st.number_input("Unit Rate (₹)", min_value=0.0, step=10.0)
                with mc8:
                    m_status = st.selectbox("Bill Status", ["Unpaid", "Partial", "Paid", "To Be Ordered"])

                if st.form_submit_button("➕ Add to Material BOQ", type="primary"):
                    if not m_name.strip():
                        st.error("Please enter the Material Name.")
                    else:
                        calc_qty = m_proc if m_proc > 0 else m_req
                        tot_cost = round(calc_qty * m_rate, 2)
                        conn = get_conn()
                        conn.execute("""
                            INSERT INTO materials (project_id, entry_date, category, material_name, unit, qty_required, qty_procured, unit_rate, total_cost, supplier_name, payment_status)
                            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                        """, (selected_project_id, str(date.today()), m_cat, m_name.strip(), m_unit, m_req, m_proc, m_rate, tot_cost, m_supp.strip(), m_status))
                        conn.commit()
                        conn.close()
                        st.success(f"Added {m_name} ({format_inr(tot_cost)})")
                        st.rerun()

            st.divider()
            st.subheader("📋 Project Material Sheet (Click Any Cell to Edit Live)")
            if not mats_df.empty:
                edited_mats = st.data_editor(
                    mats_df,
                    use_container_width=True,
                    hide_index=True,
                    num_rows="dynamic",
                    disabled=["id", "project_id"],
                    column_config={
                        "unit_rate": st.column_config.NumberColumn("Rate (₹)", format="₹%,.2f"),
                        "total_cost": st.column_config.NumberColumn("Total Cost (₹)", format="₹%,.0f"),
                        "payment_status": st.column_config.SelectboxColumn("Status", options=["Unpaid", "Partial", "Paid", "To Be Ordered"]),
                    },
                )
                if st.button("💾 Save Material Table Changes"):
                    conn = get_conn()
                    conn.execute("DELETE FROM materials WHERE project_id = ?", (selected_project_id,))
                    for _, r in edited_mats.iterrows():
                        if pd.notnull(r.get("material_name")) and str(r.get("material_name")).strip() != "":
                            qty_for_cost = float(r["qty_procured"]) if float(r["qty_procured"]) > 0 else float(r["qty_required"])
                            recalc_total = round(qty_for_cost * float(r["unit_rate"]), 2)
                            conn.execute("""
                                INSERT INTO materials (project_id, entry_date, category, material_name, unit, qty_required, qty_procured, unit_rate, total_cost, supplier_name, payment_status)
                                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                            """, (
                                selected_project_id,
                                str(r.get("entry_date") or date.today()),
                                r.get("category", "Hardware & Misc"),
                                r["material_name"],
                                r.get("unit", "Nos"),
                                float(r.get("qty_required", 0)),
                                float(r.get("qty_procured", 0)),
                                float(r.get("unit_rate", 0)),
                                recalc_total,
                                r.get("supplier_name", ""),
                                r.get("payment_status", "Unpaid"),
                            ))
                    conn.commit()
                    conn.close()
                    st.success("Material BOQ updated & totals recalculated!")
                    st.rerun()
            else:
                st.info("No materials logged for this project yet.")

        # ------------------------------------------
        # TAB 3: SITE WORK PROGRESS & MILESTONES
        # ------------------------------------------
        with tab_work:
            st.subheader("📐 Construction & Design Stage Tracker")
            st.caption("Update completion percentages directly below to recalculate overall site progress.")

            edited_miles = st.data_editor(
                miles_df[["id", "stage_name", "weightage_pct", "completion_pct", "status", "remarks"]],
                use_container_width=True,
                hide_index=True,
                num_rows="dynamic",
                disabled=["id"],
                column_config={
                    "stage_name": st.column_config.TextColumn("Work Stage / Phase", width="large"),
                    "weightage_pct": st.column_config.NumberColumn("Weightage (%)", min_value=1, max_value=100),
                    "completion_pct": st.column_config.NumberColumn("Done (%)", min_value=0, max_value=100),
                    "status": st.column_config.SelectboxColumn("Status", options=["Pending", "In Progress", "Completed", "On Hold"]),
                },
            )
            if st.button("💾 Update Work Progress", type="primary"):
                conn = get_conn()
                conn.execute("DELETE FROM milestones WHERE project_id = ?", (selected_project_id,))
                for _, r in edited_miles.iterrows():
                    if pd.notnull(r.get("stage_name")) and str(r.get("stage_name")).strip() != "":
                        comp = float(r.get("completion_pct", 0))
                        auto_status = "Completed" if comp >= 100 else ("In Progress" if comp > 0 and r.get("status") == "Pending" else r.get("status", "Pending"))
                        conn.execute("""
                            INSERT INTO milestones (project_id, stage_name, weightage_pct, completion_pct, status, remarks, updated_on)
                            VALUES (?, ?, ?, ?, ?, ?, ?)
                        """, (
                            selected_project_id,
                            r["stage_name"],
                            float(r.get("weightage_pct", 10)),
                            comp,
                            auto_status,
                            r.get("remarks", ""),
                            str(date.today()),
                        ))
                conn.commit()
                conn.close()
                st.success("Site progress updated!")
                st.rerun()

        # ------------------------------------------
        # TAB 4: PARTY-WISE KHATA
        # ------------------------------------------
        with tab_party:
            st.subheader("👷 Person & Vendor-Wise Account Summary (Khata)")
            st.caption("Total amount given to each contractor, helper, mason, or shopkeeper on this project.")

            payout_df = txns_df[txns_df["txn_type"] == "Expense / Payout"]
            if payout_df.empty:
                st.info("No payouts recorded yet.")
            else:
                khata = (
                    payout_df.groupby(["party_role", "party_name"])
                    .agg(
                        Payments_Count=("amount", "count"),
                        Total_Paid=("amount", "sum"),
                        Last_Payment_Date=("txn_date", "max"),
                    )
                    .reset_index()
                    .sort_values(by="Total_Paid", ascending=False)
                )
                st.dataframe(
                    khata,
                    use_container_width=True,
                    hide_index=True,
                    column_config={
                        "party_role": "Role",
                        "party_name": "Contractor / Helper / Shop Name",
                        "Payments_Count": "No. of Payments",
                        "Total_Paid": st.column_config.NumberColumn("Total Given (₹)", format="₹%,.0f"),
                        "Last_Payment_Date": "Last Paid On",
                    },
                )

        # ------------------------------------------
        # TAB 5: DRAWINGS & SITE INSPECTION CHECKLIST
        # ------------------------------------------
        with tab_tasks:
            st.subheader("📋 Working Drawings, Snag List & Client Approvals")
            with st.form("add_task_form", clear_on_submit=True):
                tc1, tc2, tc3, tc4 = st.columns([1.3, 2.5, 1.3, 1])
                with tc1:
                    tk_type = st.selectbox("Category", ["Drawing Issue", "Site Inspection / Snag", "Client Approval", "Material Order"])
                with tc2:
                    tk_detail = st.text_input("Action Item / Drawing / Snag *", placeholder="e.g., Issue Kitchen MEP working drawing")
                with tc3:
                    tk_assign = st.text_input("Responsible Person", placeholder="e.g., Contractor / Client")
                with tc4:
                    tk_prio = st.selectbox("Priority", ["High", "Medium", "Low"])

                if st.form_submit_button("➕ Add Action Item", type="primary"):
                    if tk_detail.strip():
                        conn = get_conn()
                        conn.execute("""
                            INSERT INTO site_tasks (project_id, created_date, task_type, task_detail, assigned_to, priority, is_done)
                            VALUES (?, ?, ?, ?, ?, ?, 0)
                        """, (selected_project_id, str(date.today()), tk_type, tk_detail.strip(), tk_assign.strip(), tk_prio))
                        conn.commit()
                        conn.close()
                        st.rerun()

            if not tasks_df.empty:
                st.divider()
                for _, row in tasks_df.iterrows():
                    col_chk, col_txt, col_del = st.columns([0.5, 8.5, 1])
                    checked = bool(row["is_done"])
                    new_chk = col_chk.checkbox("", value=checked, key=f"tsk_{row['id']}")
                    if new_chk != checked:
                        conn = get_conn()
                        conn.execute("UPDATE site_tasks SET is_done = ? WHERE id = ?", (1 if new_chk else 0, int(row["id"])))
                        conn.commit()
                        conn.close()
                        st.rerun()

                    label = f"**[{row['task_type']}]** {row['task_detail']} — *Assigned: {row['assigned_to']}* (`{row['priority']}`)"
                    if checked:
                        col_txt.markdown(f"~~{label}~~ ✅")
                    else:
                        col_txt.markdown(label)

                    if col_del.button("🗑️", key=f"del_tsk_{row['id']}"):
                        conn = get_conn()
                        conn.execute("DELETE FROM site_tasks WHERE id = ?", (int(row["id"]),))
                        conn.commit()
                        conn.close()
                        st.rerun()


# ==========================================
# VIEW 3: LIVE MATERIAL RATES & STORE LOOKUP
# ==========================================
elif nav_mode == "🛒 Live Material Rates & Store Lookup":
    st.header("🛒 Live Construction Material Rates & Store Finder")
    st.markdown(
        "Check indicative Indian market rate benchmarks or look up **live availability, specs, and prices** directly on **Amazon.in, Flipkart, and IndiaMART**."
    )

    with st.container(border=True):
        st.subheader("🔎 Instant Live Store Search (Amazon / Flipkart / IndiaMART)")
        sc1, sc2 = st.columns([3, 1])
        with sc1:
            custom_query = st.text_input(
                "Search any architectural fitting, lighting, sanitaryware, paint, or tool:",
                placeholder="e.g., Jaquar diverter, Philips 12W COB warm white, Bosch laser distance meter, Asian Paints Royale",
            )
        with sc2:
            city_filter = st.text_input("City (for IndiaMART bulk dealers)", value="Pune")

        if custom_query.strip():
            q_enc = urllib.parse.quote_plus(custom_query.strip())
            city_enc = urllib.parse.quote_plus(city_filter.strip())

            amz_url = f"https://www.amazon.in/s?k={q_enc}"
            flp_url = f"https://www.flipkart.com/search?q={q_enc}"
            ind_url = f"https://dir.indiamart.com/search.mp?ss={q_enc}&cq={city_enc}"

            b1, b2, b3 = st.columns(3)
            b1.link_button("🟠 Check Live Price on Amazon.in ↗", amz_url, use_container_width=True)
            b2.link_button("🔵 Check Live Price on Flipkart ↗", flp_url, use_container_width=True)
            b3.link_button(f"🏭 Wholesale Quotes on IndiaMART ({city_filter}) ↗", ind_url, use_container_width=True)

    st.divider()
    st.subheader("📊 Standard Civil & Interior Rate Card + Live Store Links")
    bench_df = pd.DataFrame(LIVE_MARKET_BENCHMARKS)

    cat_filter = st.multiselect(
        "Filter by Material Category",
        options=bench_df["Category"].unique().tolist(),
        default=bench_df["Category"].unique().tolist(),
    )
    filtered_bench = bench_df[bench_df["Category"].isin(cat_filter)].copy()

    filtered_bench["Amazon Link"] = filtered_bench["Amazon Query"].apply(
        lambda x: f"https://www.amazon.in/s?k={urllib.parse.quote_plus(x)}"
    )
    filtered_bench["Flipkart Link"] = filtered_bench["Flipkart Query"].apply(
        lambda x: f"https://www.flipkart.com/search?q={urllib.parse.quote_plus(x)}"
    )
    filtered_bench["IndiaMART Link"] = filtered_bench["IndiaMART Query"].apply(
        lambda x: f"https://dir.indiamart.com/search.mp?ss={urllib.parse.quote_plus(x)}"
    )

    st.dataframe(
        filtered_bench[[
            "Category",
            "Item",
            "Unit",
            "Indicative Market Rate (₹)",
            "Brands",
            "Amazon Link",
            "Flipkart Link",
            "IndiaMART Link",
        ]],
        use_container_width=True,
        hide_index=True,
        column_config={
            "Amazon Link": st.column_config.LinkColumn("Amazon.in", display_text="Search Amazon ↗"),
            "Flipkart Link": st.column_config.LinkColumn("Flipkart", display_text="Search Flipkart ↗"),
            "IndiaMART Link": st.column_config.LinkColumn("IndiaMART", display_text="Bulk Quotes ↗"),
        },
    )


# ==========================================
# VIEW 4: CREATE / MANAGE PROJECTS
# ==========================================
elif nav_mode == "➕ Create / Manage Projects":
    st.header("➕ Create a New Architectural Project")

    with st.form("new_project_form", clear_on_submit=True):
        p1, p2 = st.columns(2)
        with p1:
            p_name = st.text_input("Project Name *", placeholder="e.g., Deshmukh Residence - Nagpur")
            p_client = st.text_input("Client Name *", placeholder="e.g., Dr. Rajesh Deshmukh")
            p_loc = st.text_input("Site Location", placeholder="e.g., Dharampeth, Nagpur")
        with p2:
            p_type = st.selectbox(
                "Project Typology",
                [
                    "Residential Bungalow / Villa",
                    "Interior Design & Turnkey",
                    "Commercial Office / Retail",
                    "Apartment / Housing",
                    "Hospitality / Farmhouse",
                    "Institutional / Public",
                ],
            )
            p_budget = st.number_input("Estimated Construction Budget (₹)", min_value=0.0, step=50000.0)
            p_fee = st.number_input("Architectural Consultancy Fee (₹)", min_value=0.0, step=10000.0)

        p_notes = st.text_area("Project Scope & Design Notes", placeholder="Plot area, G+2 structure, sustainable materials, timeline...")
        add_default_stages = st.checkbox("Auto-populate standard architectural & construction stages", value=True)

        if st.form_submit_button("🏛️ Create Project Workspace", type="primary"):
            if not p_name.strip() or not p_client.strip():
                st.error("Project Name and Client Name are required.")
            else:
                try:
                    conn = get_conn()
                    c = conn.cursor()
                    c.execute("""
                        INSERT INTO projects (project_name, client_name, site_location, typology, total_budget, architect_fee, status, start_date, notes)
                        VALUES (?, ?, ?, ?, ?, ?, 'In Progress', ?, ?)
                    """, (p_name.strip(), p_client.strip(), p_loc.strip(), p_type, float(p_budget), float(p_fee), str(date.today()), p_notes.strip()))
                    new_pid = c.lastrowid

                    if add_default_stages:
                        default_stages = [
                            (new_pid, "1. Concept Design & Client Approval", 10, 0, "Pending", "", str(date.today())),
                            (new_pid, "2. Working & Structural Drawings", 15, 0, "Pending", "", str(date.today())),
                            (new_pid, "3. Foundation & Plinth Work", 20, 0, "Pending", "", str(date.today())),
                            (new_pid, "4. RCC Frame & Brick Masonry", 25, 0, "Pending", "", str(date.today())),
                            (new_pid, "5. Plaster, Flooring, Electrical & Plumbing", 15, 0, "Pending", "", str(date.today())),
                            (new_pid, "6. Finishes, Joinery & Handover", 15, 0, "Pending", "", str(date.today())),
                        ]
                        c.executemany("""
                            INSERT INTO milestones (project_id, stage_name, weightage_pct, completion_pct, status, remarks, updated_on)
                            VALUES (?, ?, ?, ?, ?, ?, ?)
                        """, default_stages)

                    conn.commit()
                    conn.close()
                    st.success(f"Project **{p_name}** created! Switch to **Project Ledger, BOQ & Site** in the sidebar to start logging.")
                except sqlite3.IntegrityError:
                    st.error("A project with that exact name already exists.")

    st.divider()
    st.subheader("Existing Projects")
    conn = get_conn()
    existing_p = pd.read_sql_query("SELECT id, project_name, client_name, site_location, typology, total_budget, status FROM projects", conn)
    conn.close()
    st.dataframe(existing_p, use_container_width=True, hide_index=True)
