from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet

# Create PDF
pdf_path = "Retail_Analysis_Report.pdf"
doc = SimpleDocTemplate(pdf_path, pagesize=A4)
styles = getSampleStyleSheet()
story = []

# Title
story.append(Paragraph("<b>Retail Business Analysis – SQL & Python Code</b>", styles['Title']))
story.append(Spacer(1, 12))

# Add sections
sections = {
    "1. Database Connection Steps": """
Step 1: Install Python packages
pip install pandas psycopg2 matplotlib seaborn

Step 2: Connect to PostgreSQL
import psycopg2
conn = psycopg2.connect(host='localhost', database='retail_db', user='your_username', password='your_password')

Step 3: Load tables into Pandas
import pandas as pd
sales = pd.read_sql("SELECT * FROM sales", conn)
inventory = pd.read_sql("SELECT * FROM inventory", conn)
""",
    "2. SQL Queries": """
-- Clean Sales Data
CREATE VIEW vw_sales_clean AS SELECT * FROM sales WHERE order_date IS NOT NULL AND product_id IS NOT NULL AND quantity > 0;

-- Calculate Revenue, Cost, Profit, Margin
SELECT product_id, category,
SUM(quantity*unit_price) AS revenue,
SUM(quantity*unit_cost) AS cost,
SUM(quantity*unit_price - quantity*unit_cost - discount) AS profit,
ROUND(100*(SUM(quantity*unit_price - quantity*unit_cost - discount)/SUM(quantity*unit_price)),2) AS margin_pct
FROM vw_sales_clean
GROUP BY product_id, category;

-- Identify Slow-Moving Items
WITH sales_90d AS (
SELECT product_id, SUM(quantity) AS qty_sold
FROM sales
WHERE order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY product_id
)
SELECT p.product_id, p.product_name, i.qty_on_hand, s.qty_sold
FROM products p
LEFT JOIN inventory i ON p.product_id=i.product_id
LEFT JOIN sales_90d s ON p.product_id=s.product_id
WHERE s.qty_sold < 10 AND i.qty_on_hand > 50;
""",
    "3. Python Code": """
# Load Data
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

sales = pd.read_sql("SELECT * FROM vw_sales_clean", conn)
inventory = pd.read_sql("SELECT * FROM inventory", conn)

# Compute Profit and Margin
sales['profit'] = (sales['quantity']*sales['unit_price']) - (sales['quantity']*sales['unit_cost']) - sales['discount']
sales['margin_pct'] = (sales['profit'] / (sales['quantity']*sales['unit_price']))*100

sku_metrics = sales.groupby(['product_id','category']).agg(
    revenue=('quantity','sum'),
    cost=('quantity','sum'),
    profit=('profit','sum'),
    margin_pct=('margin_pct','mean')
).reset_index()

# Days of Supply
sales_90 = sales[sales['order_date'] >= pd.Timestamp.today() - pd.Timedelta(days=90)]
avg_daily_sales = sales_90.groupby('product_id')['quantity'].sum()/90
days_of_supply = inventory.set_index('product_id')['qty_on_hand']/avg_daily_sales
sku_metrics = sku_metrics.merge(days_of_supply.rename('days_of_supply'), left_on='product_id', right_index=True, how='left')

# Correlation Analysis
corr = sku_metrics[['days_of_supply','margin_pct']].corr()
print(corr)

# Visualizations
sns.barplot(x='category', y='profit', data=sku_metrics)
plt.show()
sns.scatterplot(x='days_of_supply', y='margin_pct', data=sku_metrics)
plt.show()
monthly_revenue = sales.groupby(sales['order_date'].dt.to_period('M'))['profit'].sum().reset_index()
sns.lineplot(x='order_date', y='profit', data=monthly_revenue)
plt.show()
"""
}

# Add sections to PDF
for title, content in sections.items():
    story.append(Paragraph(f"<b>{title}</b>", styles['Heading2']))
    story.append(Spacer(1, 6))
    story.append(Paragraph(f"<pre>{content}</pre>", styles['Code']))
    story.append(Spacer(1, 12))

# Build PDF
doc.build(story)
print(f"PDF saved as {pdf_path}")
