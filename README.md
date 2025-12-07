# Minimal Amazon-like site (Flask)
#
# This single-file Python script bootstraps a small demo e-commerce Flask app.
# It will:
#  - create the project directory structure under the current working directory (`minimal_amazon/`)
#  - write templates and CSS files used by the app
#  - define and run the Flask app (with a small SQLite DB seeded with sample products)
#
# Running this script directly (python app.py) will start the Flask dev server on 127.0.0.1:5000.
# If you prefer, inspect the created files under `minimal_amazon/` and run them separately.

from flask import Flask, render_template, request, redirect, url_for, session, flash
from flask_sqlalchemy import SQLAlchemy
import os
import textwrap
import pathlib

# -----------------------------
# Helper: write project files
# -----------------------------
PROJECT_DIR = os.path.join(os.getcwd(), 'minimal_amazon')
TEMPLATES_DIR = os.path.join(PROJECT_DIR, 'templates')
STATIC_DIR = os.path.join(PROJECT_DIR, 'static')
IMAGES_DIR = os.path.join(STATIC_DIR, 'images')
CSS_DIR = os.path.join(STATIC_DIR, 'css')

def ensure_dirs():
    os.makedirs(TEMPLATES_DIR, exist_ok=True)
    os.makedirs(IMAGES_DIR, exist_ok=True)
    os.makedirs(CSS_DIR, exist_ok=True)


def write_file(path, content):
    with open(path, 'w', encoding='utf-8') as f:
        f.write(content)


def bootstrap_files():
    ensure_dirs()

    # app README and requirements
    write_file(os.path.join(PROJECT_DIR, 'requirements.txt'), 'Flask==2.3.2\nFlask-SQLAlchemy==3.0.3\n')
    write_file(os.path.join(PROJECT_DIR, 'README.md'), textwrap.dedent('''
        Minimal Amazon-like Flask demo

        1. Create a virtual environment: python -m venv venv
        2. Activate it and install dependencies: pip install -r requirements.txt
        3. Run this script: python app.py (it will create the project files under ./minimal_amazon and start the dev server)
        4. Open http://127.0.0.1:5000 in your browser
        ''').strip())

    # CSS
    css = textwrap.dedent('''
    /* Basic, clean layout to get you started */
    *{box-sizing:border-box}
    body{font-family:system-ui, -apple-system, 'Segoe UI', Roboto, 'Noto Sans', 'Helvetica Neue', Arial; margin:0; background:#f6f7fb}
    .site-header{display:flex; justify-content:space-between; align-items:center; padding:12px 24px; background:#131921; color:#fff}
    .brand{font-size:20px}
    .brand .accent{color:#ffd700}
    .site-header nav a{color:#fff; margin-left:14px; text-decoration:none}
    .container{max-width:1100px; margin:24px auto; padding:0 16px}
    .grid{display:grid; grid-template-columns: repeat(auto-fill,minmax(220px,1fr)); gap:16px}
    .card{background:#fff; padding:12px; border-radius:8px; box-shadow:0 2px 6px rgba(0,0,0,0.06); text-align:center}
    .card img{height:160px; object-fit:cover; width:100%; border-radius:6px}
    .price{font-weight:700; margin:8px 0}
    .btn{background:#ff9900; border:none; padding:8px 12px; border-radius:6px; cursor:pointer}
    .btn.secondary{background:#ddd}
    .product-detail{display:grid; grid-template-columns: 380px 1fr; gap:24px}
    .product-detail img{width:100%; border-radius:8px}
    .cart{width:100%; border-collapse:collapse}
    .cart th,.cart td{padding:8px; border-bottom:1px solid #eee}
    .site-footer{text-align:center; padding:18px 0; color:#666}
    .flash{background:#fff7d6; padding:8px; border-radius:6px; margin-bottom:14px}

    @media(max-width:720px){
      .product-detail{grid-template-columns:1fr}
      .grid{grid-template-columns: repeat(auto-fill,minmax(160px,1fr));}
    }
    ''')
    write_file(os.path.join(CSS_DIR, 'style.css'), css)

    # Templates
    base_html = textwrap.dedent('''
    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8" />
      <meta name="viewport" content="width=device-width,initial-scale=1" />
      <title>Mini Amazon</title>
      <link rel="stylesheet" href="/static/css/style.css" />
    </head>
    <body>
      <header class="site-header">
        <div class="brand">Mini<span class="accent">Shop</span></div>
        <nav>
          <a href="/">Home</a>
          <a href="/cart">Cart</a>
          <a href="/admin">Admin</a>
        </nav>
      </header>

      <main class="container">
        {% with messages = get_flashed_messages() %}
          {% if messages %}
            <div class="flash">
              {% for m in messages %}<div>{{ m }}</div>{% endfor %}
            </div>
          {% endif %}
        {% endwith %}

        {% block content %}{% endblock %}
      </main>

      <footer class="site-footer">
        © {{"now"|date("%Y")}} MiniShop — demo only
      </footer>
    </body>
    </html>
    ''')
    write_file(os.path.join(TEMPLATES_DIR, 'base.html'), base_html)

    index_html = textwrap.dedent('''
    {% extends 'base.html' %}
    {% block content %}
      <h1>Featured Products</h1>
      <div class="grid">
        {% for p in products %}
          <div class="card">
            <a href="{{ url_for('product_detail', pid=p.id) }}">
              <img src="/static/{{ p.image or 'images/placeholder.png' }}" alt="{{ p.title }}" />
            </a>
            <h3>{{ p.title }}</h3>
            <p class="price">₹{{ '%.2f'|format(p.price) }}</p>
            <form action="{{ url_for('add_to_cart', pid=p.id) }}" method="post">
              <input type="number" name="quantity" value="1" min="1" />
              <button type="submit" class="btn">Add to Cart</button>
            </form>
          </div>
        {% endfor %}
      </div>
    {% endblock %}
    ''')
    write_file(os.path.join(TEMPLATES_DIR, 'index.html'), index_html)

    product_html = textwrap.dedent('''
    {% extends 'base.html' %}
    {% block content %}
      <div class="product-detail">
        <img src="/static/{{ product.image or 'images/placeholder.png' }}" alt="{{ product.title }}">
        <div class="meta">
          <h1>{{ product.title }}</h1>
          <p class="price">₹{{ '%.2f'|format(product.price) }}</p>
          <p>{{ product.description }}</p>

          <form action="{{ url_for('add_to_cart', pid=product.id) }}" method="post">
            <label>Quantity: <input type="number" name="quantity" value="1" min="1"></label>
            <button class="btn" type="submit">Add to cart</button>
          </form>
        </div>
      </div>
    {% endblock %}
    ''')
    write_file(os.path.join(TEMPLATES_DIR, 'product.html'), product_html)

    cart_html = textwrap.dedent('''
    {% extends 'base.html' %}
    {% block content %}
      <h1>Your Cart</h1>
      <form action="{{ url_for('update_cart') }}" method="post">
        <table class="cart">
          <thead><tr><th>Product</th><th>Qty</th><th>Subtotal</th></tr></thead>
          <tbody>
            {% for item in items %}
              <tr>
                <td>{{ item.product.title }}</td>
                <td><input type="number" name="qty_{{ item.product.id }}" value="{{ item.qty }}" min="0"></td>
                <td>₹{{ '%.2f'|format(item.subtotal) }}</td>
              </tr>
            {% endfor %}
          </tbody>
        </table>
        <div class="cart-summary">Total: ₹{{ '%.2f'|format(total) }}</div>
        <div class="cart-actions">
          <button type="submit" class="btn">Update Cart</button>
          <button type="button" class="btn secondary" onclick="alert('Checkout demo — integrate payments to go live')">Proceed to Checkout</button>
        </div>
      </form>
    {% endblock %}
    ''')
    write_file(os.path.join(TEMPLATES_DIR, 'cart.html'), cart_html)

    admin_html = textwrap.dedent('''
    {% extends 'base.html' %}
    {% block content %}
      <h1>Admin — Add Product</h1>
      <form method="post">
        <label>Title<br><input name="title" required></label><br>
        <label>Price (number)<br><input name="price" required></label><br>
        <label>Image path (under static/)<br><input name="image" placeholder="images/ring1.jpg"></label><br>
        <label>Description<br><textarea name="description"></textarea></label><br>
        <button type="submit" class="btn">Add Product</button>
      </form>

      <h2>Existing Products</h2>
      <ul>
        {% for p in products %}
          <li>{{ p.id }} — {{ p.title }} — ₹{{ '%.2f'|format(p.price) }}</li>
        {% endfor %}
      </ul>
    {% endblock %}
    ''')
    write_file(os.path.join(TEMPLATES_DIR, 'admin.html'), admin_html)

    # placeholder image
    placeholder = (
        """data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='600' height='400'>"
        "<rect fill='%23f3f4f6' width='100%' height='100%'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle'"
        " font-size='20' fill='%23999'>Placeholder</text></svg>"""
    )
    # store as .svg file so browsers can display it
    write_file(os.path.join(IMAGES_DIR, 'placeholder.svg'), """<svg xmlns='http://www.w3.org/2000/svg' width='600' height='400'><rect fill='#f3f4f6' width='100%' height='100%'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle' font-size='20' fill='#999'>Placeholder</text></svg>""")

    print(f"Bootstrapped project files under: {PROJECT_DIR}")

# -----------------------------
# Flask app (identical behavior to the original app.py)
# -----------------------------

# NOTE: We point Flask's template and static folder to the project subdirectory
app = Flask(__name__, template_folder=TEMPLATES_DIR, static_folder=STATIC_DIR)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(PROJECT_DIR, 'store.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['SECRET_KEY'] = 'dev-secret-key-change-me'

db = SQLAlchemy(app)

class Product(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(120), nullable=False)
    description = db.Column(db.Text, nullable=True)
    price = db.Column(db.Float, nullable=False)
    image = db.Column(db.String(200), nullable=True)  # relative path under static

    def to_dict(self):
        return {
            'id': self.id,
            'title': self.title,
            'description': self.description,
            'price': self.price,
            'image': self.image,
        }


@app.before_first_request
def create_tables_and_seed():
    # ensure directories/files exist before creating DB
    bootstrap_files()
    db.create_all()
    if Product.query.count() == 0:
        sample = [
            Product(title='Elegant Gold Ring', description='18k gold ring with subtle shine', price=1299.00, image='images/placeholder.svg'),
            Product(title='Silver Band', description='Classic silver band for everyday wear', price=499.00, image='images/placeholder.svg'),
            Product(title='Diamond Pendant', description='Small diamond pendant on chain', price=3499.00, image='images/placeholder.svg'),
        ]
        db.session.bulk_save_objects(sample)
        db.session.commit()


# helpers for cart stored in server-side session
def get_cart():
    return session.setdefault('cart', {})


@app.route('/')
def index():
    products = Product.query.all()
    return render_template('index.html', products=products)


@app.route('/product/<int:pid>')
def product_detail(pid):
    p = Product.query.get_or_404(pid)
    return render_template('product.html', product=p)


@app.route('/add-to-cart/<int:pid>', methods=['POST'])
def add_to_cart(pid):
    try:
        qty = int(request.form.get('quantity', 1))
    except (ValueError, TypeError):
        qty = 1
    cart = get_cart()
    cart[str(pid)] = cart.get(str(pid), 0) + qty
    session['cart'] = cart
    flash('Added to cart')
    return redirect(request.referrer or url_for('index'))


@app.route('/cart')
def view_cart():
    cart = get_cart()
    items = []
    total = 0.0
    for pid_str, qty in cart.items():
        p = Product.query.get(int(pid_str))
        if p:
            subtotal = p.price * qty
            total += subtotal
            items.append({'product': p, 'qty': qty, 'subtotal': subtotal})
    return render_template('cart.html', items=items, total=total)


@app.route('/update-cart', methods=['POST'])
def update_cart():
    cart = get_cart()
    for key, value in request.form.items():
        if key.startswith('qty_'):
            pid = key.split('_', 1)[1]
            try:
                q = int(value)
            except ValueError:
                q = 0
            if q <= 0:
                cart.pop(pid, None)
            else:
                cart[pid] = q
    session['cart'] = cart
    flash('Cart updated')
    return redirect(url_for('view_cart'))


# simple admin to add product (no auth for simplicity — add auth before production)
@app.route('/admin', methods=['GET', 'POST'])
def admin():
    if request.method == 'POST':
        title = request.form['title']
        description = request.form.get('description')
        try:
            price = float(request.form.get('price', 0))
        except (ValueError, TypeError):
            price = 0.0
        image = request.form.get('image')  # expect path under static/ e.g. images/ring1.jpg
        p = Product(title=title, description=description, price=price, image=image)
        db.session.add(p)
        db.session.commit()
        flash('Product added')
        return redirect(url_for('admin'))
    products = Product.query.order_by(Product.id.desc()).all()
    return render_template('admin.html', products=products)


# -----------------------------
# Basic self-tests (run once before starting server)
# -----------------------------

def run_smoke_tests():
    print('\nRunning basic smoke tests using Flask test client...')
    client = app.test_client()
    ok = True
    try:
        r = client.get('/')
        assert r.status_code == 200
        print('GET / -> 200 OK')

        r = client.get('/admin')
        assert r.status_code == 200
        print('GET /admin -> 200 OK')

    except AssertionError as e:
        ok = False
        print('Smoke test failed:', e)
    if ok:
        print('All smoke tests passed.')
    else:
        print('Some smoke tests failed. Check logs above.')


# -----------------------------
# Run app
# -----------------------------
if __name__ == '__main__':
    # Bootstrap files and run local tests before starting
    bootstrap_files()
    # ensure DB folder exists
    pathlib.Path(PROJECT_DIR).mkdir(parents=True, exist_ok=True)

    # initialize DB (will seed on first request due to @before_first_request)
    # run quick smoke tests to confirm templates render
    run_smoke_tests()

    # Start Flask development server
    print('\nStarting Flask development server at http://127.0.0.1:5000 (use Ctrl+C to stop)')
    app.run(debug=True)
    
