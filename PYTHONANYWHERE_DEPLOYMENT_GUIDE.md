# PythonAnywhere Deployment Guide

This guide provides the exact steps required to perfectly host your Django web application on PythonAnywhere, ensuring everything works seamlessly (especially the database and static files).

## 1. Preparing the Django Project (`settings.py`)

Before pushing your code to GitHub, you must modify your `settings.py` file to handle production routing and static files:

1. **Set Allowed Hosts:** Tell Django what domain name it is allowed to run on.
   ```python
   ALLOWED_HOSTS = ['yourusername.pythonanywhere.com', 'localhost', '127.0.0.1']
   ```

2. **Allow HTTPS Cross-Origin Requests:** If your frontend or mobile app communicates with APIs, ensure HTTPS origins are trusted.
   ```python
   CSRF_TRUSTED_ORIGINS = ['https://yourusername.pythonanywhere.com']
   ```

3. **Configure Static Files:** This is critical for CSS, JS, and Images to load. Ensure you have the **leading slash** in `STATIC_URL`, and define `STATIC_ROOT`.
   ```python
   STATIC_URL = '/static/'
   STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
   STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
   ```

## 2. Pushing Code & Logging into PythonAnywhere

1. Push all your changes to your GitHub repository.
2. Log into PythonAnywhere.
3. Open a new **Bash Console**.

## 3. Cloning Code & Setting Up Environment

In the Bash Console, run the following commands sequentially:

```bash
# 1. Clone your code from GitHub (replace with your repo URL)
git clone https://github.com/YourUsername/Your_Repo_Name.git

# 2. Create a virtual environment (replace myenv with your chosen name)
mkvirtualenv myenv --python=python3.10

# 3. Navigate into your project folder
cd Your_Repo_Name/mysite

# 4. Install requirements (assuming you have a requirements.txt)
pip install -r requirements.txt
```
*(Note: If you don't have a requirements.txt, manually `pip install django djangorestframework django-cors-headers` etc).*

## 4. Database Initialization & Static Files Collection

While still inside your `mysite` folder (where `manage.py` lives), run these commands:

```bash
# 1. Prepare database tables
python manage.py makemigrations
python manage.py migrate

# 2. Create an admin account
python manage.py createsuperuser

# 3. Collect Static Files (CRITICAL STEP for CSS/JS to work)
python manage.py collectstatic
```
When it asks if you want to overwrite static files, type `yes` and press Enter.

## 5. Web Tab Configuration on PythonAnywhere

Now click on the **Web** tab at the top right of the screen and follow these exact steps:

1. Click **Add a new web app**.
2. Select **Manual configuration** (Do NOT choose Django auto-config).
3. Select **Python 3.10**.

### 5a. Virtualenv Configuration
Scroll down to the **Virtualenv** section. Enter the name of your virtual environment:
`/home/yourusername/.virtualenvs/myenv`

### 5b. Code Configuration & WSGI
Scroll up to the **Code** section.
1. Set the **Source code** directory to `/home/yourusername/Your_Repo_Name/mysite`
2. Click the link next to **WSGI configuration file**.
3. Delete **EVERYTHING** inside that file and paste this:

```python
import os
import sys

# Assuming your repository folder is named "Your_Repo_Name"
path = '/home/yourusername/Your_Repo_Name/mysite'
if path not in sys.path:
    sys.path.append(path)

# Ensure this matches your project folder name where settings.py lives
os.environ['DJANGO_SETTINGS_MODULE'] = 'mysite.settings'

# Import Django WSGI handler
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```
Click **Save** and go back to the Web tab.

### 5c. Static Files Mapping (CRITICAL)
Scroll down to the **Static files** section. You must add a mapping here so PythonAnywhere's Nginx server knows where your `collectstatic` files went.
- **URL:** `/static/`
- **Directory:** `/home/yourusername/Your_Repo_Name/mysite/staticfiles/`

*(Make sure you enter `staticfiles` and not just `static`, matching your `STATIC_ROOT` from `settings.py`).*

## 6. Reload and Verify

1. Scroll to the top of the Web tab and click the big green **Reload** button.
2. Click your web app URL (e.g., `https://yourusername.pythonanywhere.com`).
3. Your website should load instantly with all CSS and Javascript working perfectly!
4. Navigate to `/admin/` and log in with your superuser credentials to verify database functionality.
