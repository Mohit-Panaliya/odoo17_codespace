
# 📝 Odoo 17 Development Environment README

This document outlines the **Three-Script Architecture** for running Odoo 17 Community Edition on Python 3.10 within GitHub Codespaces. This setup ensures hardware stability and environment isolation.

## 📁 Project Structure

```text
/workspaces/odooCodesapce
├── odoo/               # Core Odoo 17 Source Code
├── odoo-venv/          # Isolated Python 3.10 Environment
├── custom_addons/      # Your custom development folder
├── odoo.conf           # Server configuration file
├── setup_env.sh        # Script 1: Infrastructure setup
├── install_deps.sh     # Script 2: Dependency installation
└── run.sh              # Script 3: Daily execution & service check

```

---

## 🛠️ Phase 1: Installation (First Time Only)

### 1. Setup Infrastructure

This script installs `pyenv`, compiles Python 3.10, clones the Odoo source, and creates the virtual environment.

```bash
chmod +x *.sh
./setup_env.sh

```

### 2. Install Dependencies

Activate the environment and run the sanitized dependency installer. This script patches `requirements.txt` to fix `gevent`, `greenlet`, and `setuptools` version conflicts.

```bash
source odoo-venv/bin/activate
./install_deps.sh

```

---

## 🚀 Phase 2: Daily Workflow

### 1. Launching the Server

The `run.sh` script is idempotent. It checks if PostgreSQL is running, kills any "ghost" Odoo processes on port 8069, and starts the server using the config file.

```bash
./run.sh

```

### 2. Accessing Odoo

Once the logs show `HTTP service at 0.0.0.0:8069`, click the **"Open in Browser"** popup from Codespaces.

* **Database:** `odoo_dev` (Create on first run)
* **Email:** `admin`
* **Password:** `admin`

---

## 👨‍💻 Phase 3: Developer Commands

### Creating a New Module (Scaffolding)

In Odoo, we use the `scaffold` command to generate the folder structure (Models, Views, Controllers).

```bash
./odoo-venv/bin/python odoo/odoo-bin scaffold my_module custom_addons

```

### Updating Modules

If you modify Python code or XML views, restart the server with the update flag:

```bash
./run.sh -u my_module

```

---

## 📦 Script Contents (For Reference)

### **setup_env.sh**

```bash
#!/bin/bash
set -e
WORKING_DIR=$(pwd)
PYTHON_VER="3.10.13"

echo "📂 Initializing Odoo 17 Infrastructure..."
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
if ! command -v pyenv &> /dev/null; then
    curl https://pyenv.run | bash
    eval "$(pyenv init -)"
fi
pyenv install $PYTHON_VER -s
pyenv local $PYTHON_VER

if [ ! -d "odoo" ]; then
    git clone https://www.github.com/odoo/odoo --depth 1 --branch 17.0 odoo
fi

if [ ! -d "odoo-venv" ]; then
    $(pyenv root)/versions/$PYTHON_VER/bin/python -m venv odoo-venv
fi

if [ ! -f "odoo.conf" ]; then
    cat <<EOF > odoo.conf
[options]
admin_passwd = admin
db_host = localhost
db_user = odoo
db_password = odoo
db_port = 5432
addons_path = $WORKING_DIR/odoo/addons,$WORKING_DIR/custom_addons
EOF
fi
mkdir -p custom_addons
echo "✅ Infrastructure ready."

```

### **install_deps.sh**

```bash
#!/bin/bash
if [[ "$VIRTUAL_ENV" != *"odoo-venv"* ]]; then
    echo "❌ ERROR: Activate venv first: source odoo-venv/bin/activate"
    exit 1
fi

echo "🆙 Installing Odoo 17 Dependencies..."
pip install --upgrade pip setuptools==67.8.0 wheel

cat <<EOF > odoo/requirements.txt
Babel==2.10.3
chardet==4.0.0
cryptography==42.0.8
decorator==5.1.1
docutils==0.17
ebaysdk==2.1.5
freezegun==1.2.1
geoip2==2.9.0
gevent==23.9.1
greenlet==3.3.1
idna==3.6
Jinja2==3.1.2
libsass==0.22.0
lxml==5.2.1
lxml-html-clean==0.4.3
MarkupSafe==2.1.5
num2words==0.5.13
ofxparse==0.21
passlib==1.7.4
Pillow==10.2.0
polib==1.1.1
psutil==5.9.8
psycopg2-binary==2.9.11
pydot==1.4.2
pyopenssl==24.1.0
pyparsing==3.1.2
PyPDF2==2.12.1
python-dateutil==2.8.2
python-ldap==3.4.4
python-stdnum==1.19
pytz==2024.1
pyusb==1.2.1
qrcode==7.4.2
reportlab==4.1.0
requests==2.31.0
urllib3==1.26.15
vobject==0.9.6.1
Werkzeug==3.0.1
XlsxWriter==3.1.9
xlrd==2.0.1
xlwt==1.3.0
zeep==4.2.1
zope.event==5.0
zope.interface==6.2
rjsmin==1.2.0
EOF

pip install -r odoo/requirements.txt
echo "🚀 Dependencies installed successfully!"

```

### **run.sh**

```bash
#!/bin/bash
echo "🐘 Checking PostgreSQL..."
sudo service postgresql status > /dev/null || sudo service postgresql start
sudo -u postgres psql -c "CREATE USER odoo WITH PASSWORD 'odoo' SUPERUSER;" 2>/dev/null || true

echo "🌐 Checking Port 8069..."
OS_PID=$(lsof -t -i:8069)
if [ ! -z "$OS_PID" ]; then
    echo "⚠️ Odoo already running (PID: $OS_PID). Restarting..."
    kill -9 $OS_PID
fi

echo "🚀 Launching Odoo 17..."
./odoo-venv/bin/python odoo/odoo-bin -c odoo.conf "$@"
