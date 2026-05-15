# PythonAnywhere Deployment Guide (Easy Transport System)

This guide provides the exact steps to host this repository on PythonAnywhere. It has been updated to reflect the nested project structure and the recent tracking fixes.

## 1. Preparing the Django Project (`settings.py`)

Ensure your `mysite/mysite/settings.py` has these settings before pushing:

1. **Allowed Hosts:**
   ```python
   ALLOWED_HOSTS = ['yourusername.pythonanywhere.com', 'localhost', '127.0.0.1']
   ```

2. **CSRF Trusted Origins:**
   ```python
   CSRF_TRUSTED_ORIGINS = ['https://yourusername.pythonanywhere.com']
   ```

3. **Static Files:**
   ```python
   STATIC_URL = '/static/'
   STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
   STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
   ```

## 2. Cloning & Environment Setup

In your PythonAnywhere **Bash Console**:

```bash
# 1. Clone the repository
git clone https://github.com/Maruf2103/Easy_way_trial.git

# 2. Create and activate virtual environment
mkvirtualenv myenv --python=python3.10

# 3. Install dependencies
cd Easy_way_trial/mysite
pip install -r requirements.txt
```

## 3. Database & Static Files

While inside the `mysite` folder (the one containing `manage.py`):

```bash
# 1. Setup Database
python manage.py makemigrations
python manage.py migrate

# 2. Create Admin Account
python manage.py createsuperuser

# 3. Collect Static Files (Crucial for CSS/JS)
python manage.py collectstatic
```

## 4. Web Tab Configuration

1. Create a **New Web App** -> **Manual Configuration** -> **Python 3.10**.
2. **Virtualenv**: Set path to `/home/yourusername/.virtualenvs/myenv`.
3. **Static Files Section**:
   - **URL**: `/static/`
   - **Directory**: `/home/yourusername/Easy_way_trial/mysite/staticfiles/`

### 4a. WSGI Configuration
Click the **WSGI configuration file** link and replace everything with this:

```python
import os
import sys

# Path to the folder containing manage.py
path = '/home/yourusername/Easy_way_trial/mysite'
if path not in sys.path:
    sys.path.append(path)

os.environ['DJANGO_SETTINGS_MODULE'] = 'mysite.settings'

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

## 5. Mobile App Setup (Flutter)

If you are using the tracking feature:
1. **Permissions**: Ensure `android/app/src/main/AndroidManifest.xml` includes:
   `<uses-permission android:name="android.permission.INTERNET" />`
2. **API URL**: In `lib/main.dart`, update the URL to your PythonAnywhere domain:
   `https://yourusername.pythonanywhere.com/api/bus/1/update/`
3. **Bus ID**: Ensure you have created a **Bus** with **ID 1** in your Django Admin panel and marked it as **Active**.

## 6. Reload & Verify
1. Click **Reload** on the Web tab.
2. Visit `https://yourusername.pythonanywhere.com/admin/` to verify the database.
3. Check the **Error Log** if the tracking map doesn't show your location.
