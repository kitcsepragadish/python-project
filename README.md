
from flask import Flask, render_template, request, redirect, url_for, jsonify, send_file
import sqlite3
from datetime import datetime
import csv
import os
import io

app = Flask(__name__)
DB_NAME = 'database.db'

def get_db_connection():
    conn = sqlite3.connect(DB_NAME)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db_connection()
    conn.execute('''
        CREATE TABLE IF NOT EXISTS sales (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            product_name TEXT NOT NULL,
            quantity INTEGER NOT NULL,
            price REAL NOT NULL,
            total REAL NOT NULL,
            date TEXT NOT NULL
        )
    ''')
    conn.commit()
    conn.close()

@app.route('/')
def dashboard():
    conn = get_db_connection()
    
    # Get total revenue
    total_revenue_query = conn.execute("SELECT SUM(total) as revenue FROM sales").fetchone()
    total_revenue = total_revenue_query['revenue'] if total_revenue_query['revenue'] else 0

    # Get total products sold
    products_sold_query = conn.execute("SELECT SUM(quantity) as count FROM sales").fetchone()
    total_products = products_sold_query['count'] if products_sold_query['count'] else 0

    # Get today's sales
    today = datetime.now().strftime('%Y-%m-%d')
    daily_sales_query = conn.execute("SELECT SUM(total) as daily_total FROM sales WHERE date = ?", (today,)).fetchone()
    daily_sales = daily_sales_query['daily_total'] if daily_sales_query['daily_total'] else 0

    # Get monthly sales
    current_month = datetime.now().strftime('%Y-%m')
    monthly_sales_query = conn.execute("SELECT SUM(total) as monthly_total FROM sales WHERE strftime('%Y-%m', date) = ?", (current_month,)).fetchone()
    monthly_sales = monthly_sales_query['monthly_total'] if monthly_sales_query['monthly_total'] else 0

    # Top 5 products
    top_products = conn.execute('''
        SELECT product_name, SUM(total) as total_sales, SUM(quantity) as total_qty
        FROM sales
        GROUP BY product_name
        ORDER BY total_sales DESC
        LIMIT 5
    ''').fetchall()

    conn.close()
    return render_template('dashboard.html', 
                           total_revenue=total_revenue, 
                           total_products=total_products,
                           daily_sales=daily_sales,
                           monthly_sales=monthly_sales,
                           top_products=top_products)

@app.route('/sales')
def sales_list():
    query_search = request.args.get('search', '')
    start_date = request.args.get('start_date', '')
    end_date = request.args.get('end_date', '')

    query = "SELECT * FROM sales WHERE 1=1"
    params = []

    if query_search:
        query += " AND product_name LIKE ?"
        params.append(f"%{query_search}%")
    
    if start_date:
        query += " AND date >= ?"
        params.append(start_date)
        
    if end_date:
        query += " AND date <= ?"
        params.append(end_date)
        
    query += " ORDER BY date DESC"

    conn = get_db_connection()
    sales = conn.execute(query, params).fetchall()
    conn.close()
    
    return render_template('sales.html', sales=sales, search=query_search, start_date=start_date, end_date=end_date)

@app.route('/sales/add', methods=('GET', 'POST'))
def add_sale():
    if request.method == 'POST':
        product_name = request.form['product_name']
        quantity = int(request.form['quantity'])
        price = float(request.form['price'])
        date = request.form['date']
        total = quantity * price

        conn = get_db_connection()
        conn.execute('INSERT INTO sales (product_name, quantity, price, total, date) VALUES (?, ?, ?, ?, ?)',
                     (product_name, quantity, price, total, date))
        conn.commit()
        conn.close()
        return redirect(url_for('sales_list'))

    return render_template('add_sale.html')

@app.route('/sales/edit/<int:id>', methods=('GET', 'POST'))
def edit_sale(id):
    conn = get_db_connection()
    sale = conn.execute('SELECT * FROM sales WHERE id = ?', (id,)).fetchone()

    if request.method == 'POST':
        product_name = request.form['product_name']
        quantity = int(request.form['quantity'])
        price = float(request.form['price'])
        date = request.form['date']
        total = quantity * price

        conn.execute('''
            UPDATE sales
            SET product_name = ?, quantity = ?, price = ?, total = ?, date = ?
            WHERE id = ?
        ''', (product_name, quantity, price, total, date, id))
        conn.commit()
        conn.close()
        return redirect(url_for('sales_list'))

    conn.close()
    return render_template('edit_sale.html', sale=sale)

@app.route('/sales/delete/<int:id>', methods=('POST',))
def delete_sale(id):
    conn = get_db_connection()
    conn.execute('DELETE FROM sales WHERE id = ?', (id,))
    conn.commit()
    conn.close()
    return redirect(url_for('sales_list'))

@app.route('/api/chart-data')
def chart_data():
    conn = get_db_connection()
    
    # Daily sales (Last 7 days)
    daily_sales = conn.execute('''
        SELECT date, SUM(total) as total 
        FROM sales 
        GROUP BY date 
        ORDER BY date DESC LIMIT 7
    ''').fetchall()
    
    # Monthly sales (Current year)
    monthly_sales = conn.execute('''
        SELECT strftime('%Y-%m', date) as month, SUM(total) as total 
        FROM sales 
        GROUP BY month 
        ORDER BY month
    ''').fetchall()
    
    # Product sales (Pie chart)
    product_sales = conn.execute('''
        SELECT product_name, SUM(total) as total 
        FROM sales 
        GROUP BY product_name
    ''').fetchall()
    
    conn.close()
    
    # Format data for Chart.js
    daily_sales.reverse() # chronologically
    
    return jsonify({
        'daily': {
            'labels': [row['date'] for row in daily_sales],
            'data': [row['total'] for row in daily_sales]
        },
        'monthly': {
            'labels': [row['month'] for row in monthly_sales],
            'data': [row['total'] for row in monthly_sales]
        },
        'products': {
            'labels': [row['product_name'] for row in product_sales],
            'data': [row['total'] for row in product_sales]
        }
    })

@app.route('/export/csv')
def export_csv():
    conn = get_db_connection()
    sales = conn.execute('SELECT * FROM sales ORDER BY date DESC').fetchall()
    conn.close()

    si = io.StringIO()
    cw = csv.writer(si)
    cw.writerow(['ID', 'Product Name', 'Quantity', 'Price', 'Total', 'Date'])
    
    for row in sales:
        cw.writerow([row['id'], row['product_name'], row['quantity'], row['price'], row['total'], row['date']])
        
    output = io.BytesIO()
    output.write(si.getvalue().encode('utf-8'))
    output.seek(0)
    
    return send_file(output, mimetype='text/csv', as_attachment=True, download_name='sales_data.csv')

if __name__ == '__main__':
    init_db()
    app.run(debug=True)
