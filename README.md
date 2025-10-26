# loyal
micro finance 
# app.py
import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from io import BytesIO
import openpyxl
from openpyxl.drawing.image import Image
from datetime import datetime

# === PAGE CONFIG ===
st.set_page_config(
    page_title="Microfinance KPI Dashboard",
    page_icon="moneybag",
    layout="wide"
)

# === TITLE ===
st.title("Microfinance KPI Dashboard")
st.markdown("**Upload daily loan data → Get weekly/monthly reports + Excel with charts**")

# === LOAD DATA ===
@st.cache_data
def load_data(file):
    df = pd.read_csv(file)
    df['date'] = pd.to_datetime(df['date'])
    df['week'] = df['date'].dt.isocalendar().week
    df['month'] = df['date'].dt.to_period('M').astype(str)
    return df

# === UPLOAD ===
uploaded_file = st.file_uploader("Upload your CSV file", type="csv")

if uploaded_file:
    df = load_data(uploaded_file)
    st.success(f"Loaded {len(df):,} records")

    # === FILTERS ===
    col1, col2, col3 = st.columns(3)
    with col1:
        branches = st.multiselect("Branch", df['branch'].unique(), df['branch'].unique())
    with col2:
        employees = st.multiselect("Employee", df['employee'].unique(), df['employee'].unique())
    with col3:
        period = st.radio("View", ["Weekly", "Monthly"])

    start_date = st.date_input("Start Date", df['date'].min())
    end_date = st.date_input("End Date", df['date'].max())

    # Apply filters
    mask = (
        df['branch'].isin(branches) &
        df['employee'].isin(employees) &
        (df['date'] >= pd.to_datetime(start_date)) &
        (df['date'] <= pd.to_datetime(end_date))
    )
    filtered = df[mask].copy()

    if filtered.empty:
        st.warning("No data after filtering.")
    else:
        # === GROUP BY PERIOD ===
        if period == "Weekly":
            group_cols = ['branch', 'employee', 'week']
            filtered['period'] = filtered['week'].astype(str)
        else:
            group_cols = ['branch', 'employee', 'month']
            filtered['period'] = filtered['month']

        summary = filtered.groupby(group_cols).agg({
            'profit': 'sum',
            'target_profit': 'first',
            'outstanding': 'mean',
            'loan_steps_maintained': 'mean',
            'investment': 'sum',
            'loan_amount': 'sum'
        }).reset_index()

        summary['achievement_%'] = (summary['profit'] / summary['target_profit'] * 100).round(1)

        # Company totals
        company = summary.groupby('period').agg({
            'profit': 'sum',
            'target_profit': 'sum',
            'outstanding': 'mean',
            'loan_steps_maintained': 'mean'
        }).reset_index()
        company['achievement_%'] = (company['profit'] / company['target_profit'] * 100).round(1)

        # === DISPLAY TABLES ===
        st.subheader(f"{period} Performance by Employee")
        st.dataframe(summary.style.format({
            'profit': '₹{:.0f}',
            'target_profit': '₹{:.0f}',
            'outstanding': '₹{:.0f}',
            'investment': '₹{:.0f}',
            'loan_amount': '₹{:.0f}',
            'achievement_%': '{:.1f}%'
        }))

        st.subheader("Company Overview")
        st.dataframe(company.style.format({
            'profit': '₹{:.0f}',
            'target_profit': '₹{:.0f}',
            'achievement_%': '{:.1f}%'
        }))

        # === CHARTS ===
        fig1, ax1 = plt.subplots(figsize=(10, 6))
        sns.barplot(data=summary, x='achievement_%', y='employee', hue='branch', ax=ax1, palette="viridis")
        ax1.set_title(f"{period} Achievement % by Employee")
        ax1.set_xlabel("Achievement %")
        st.pyplot(fig1)

        fig2, ax2 = plt.subplots(figsize=(10, 6))
        company.plot(x='period', y=['profit', 'target_profit'], kind='line', marker='o', ax=ax2)
        ax2.set_title("Company Profit vs Target")
        ax2.set_ylabel("Amount (₹)")
        st.pyplot(fig2)

        # === EXCEL EXPORT ===
        def create_excel():
            output = BytesIO()
            with pd.ExcelWriter(output, engine='openpyxl') as writer:
                filtered.to_excel(writer, sheet_name='Raw Data', index=False)
                summary.to_excel(writer, sheet_name=f'{period} Summary', index=False)
                company.to_excel(writer, sheet_name='Company Totals', index=False)

                wb = writer.book
                ws = wb.create_sheet("Charts")

                # Chart 1
                img1 = BytesIO()
                fig1.savefig(img1, format='png', bbox_inches='tight', dpi=150)
                img1.seek(0)
                ws.add_image(Image(img1), "A1")

                # Chart 2
                img2 = BytesIO()
                fig2.savefig(img2, format='png', bbox_inches='tight', dpi=150)
                img2.seek(0)
                ws.add_image(Image(img2), "A30")

            output.seek(0)
            return output

        excel_data = create_excel()

        st.download_button(
            label="Download Full Excel Report (with Charts)",
            data=excel_data,
            file_name=f"Microfinance_KPI_{period}_{datetime.now().strftime('%Y%m%d')}.xlsx",
            mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
        )

else:
    st.info("**Upload a CSV to begin**")
    st.markdown("""
    ### Required Columns:
date, branch, employee, loan_amount, outstanding, profit, target_profit, investment, loan_steps_maintained
text""")

2. requirements.txt
txtstreamlit
pandas
matplotlib
seaborn
openpyxl
pillow

3. sample_data.csv – Example Data
csvdate,branch,employee,loan_amount,outstanding,profit,target_profit,investment,loan_steps_maintained
2025-01-01,Branch A,John Doe,5000,45000,1200,1000,2000,95
2025-01-02,Branch A,Jane Smith,3000,46000,800,1000,1500,90
2025-01-08,Branch B,Alice Lee,7000,52000,1800,1500,3000,98
2025-01-15,Branch A,John Doe,6000,48000,1500,1000,2500,92
2025-02-01,Branch B,Alice Lee,8000,55000,2000,1500,3500,97
2025-02-10,Branch A,Jane Smith,4000,49000,1000,1000,1800,88
2025-02-15,Branch B,Bob Kumar,9000,60000,2200,1800,4000,99

4. .streamlit/config.toml – Make It Look Pro
toml[theme]
primaryColor = "#00C853"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F5F5F5"
textColor = "#212121"
font = "sans serif"

[server]
enableCORS = false
enableXsrfProtection = false

5. README.md – For Your Team
md# Microfinance KPI Dashboard

Live App: [https://yourname-microfinance-dashboard.streamlit.app](https://yourname-microfinance-dashboard.streamlit.app)

## How to Use
1. Click the link above
2. Upload your daily CSV
3. Filter by branch/employee/date
4. Click **Download Excel** → get report with charts

## Required CSV Columns
date, branch, employee, loan_amount, outstanding, profit, target_profit, investment, loan_steps_maintained
text## For Owner
- To update: Edit `app.py` → push to GitHub → auto-deploys

HOW TO PUSH TO GITHUB (2 MINUTES)
Step 1: Open Terminal (or CMD)
bash# Go to your folder
cd microfinance-dashboard

# Initialize git
git init
git add .
git commit -m "First version of Microfinance KPI app"

# Create repo on GitHub (already done), then:
git remote add origin https://github.com/YOURUSERNAME/microfinance-dashboard.git
git branch -M main
git push -u origin main

