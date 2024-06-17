# Vehicle-Verification-System
#Vehicle checker for student that login in school premises.

import datetime
import sys
import mysql.connector
from PyQt5 import QtWidgets
from admin import Ui_MainW2
from admin_display import Ui_Dialog5
from inbox import Ui_Dialog10
from input_login import Ui_Dialog2
from license_expirations import Ui_Dialog9
from login import Ui_MainWindow
from orcr_expirations import Ui_Dialog8
from plate_numbers import Ui_Dialog7
from username_password import Ui_Dialog6
from welcome import Ui_Dialog3
from history import Ui_Dialog


class Window(QtWidgets.QMainWindow, Ui_MainWindow):
    def __init__(self):
        super(Window, self).__init__()
        self.ui = Ui_MainWindow()
        self.ui.setupUi(self)
        self.ui.pushButton_3.clicked.connect(self.input_login)
        self.current_window = None
        self.logged_in_user = None

    def close_current_window(self):
        if self.current_window:
            self.current_window.close()

    def input_login(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_input_log = Ui_Dialog2()
        self.ui_input_log.setupUi(self.current_window)
        self.current_window.show()
        self.hide()
        self.current_window.activateWindow()
        self.ui_input_log.pushButton_2.clicked.connect(self.login_student)
        self.ui_input_log.pushButton_4.clicked.connect(self.admin)

    def welcome(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_welcome = Ui_Dialog3()
        self.ui_welcome.setupUi(self.current_window)
        self.current_window.show()
        self.ui_welcome.pushButton_2.clicked.connect(self.input_login)
        current_datetime = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        self.ui_welcome.textBrowser.setPlainText("Current Date and Time:\n" + current_datetime)
        self.display_user_info()
        self.check_orcr_expiry()
        self.check_license_expiry()

    def check_orcr_expiry(self):
        try:
            connection = self.db_connect()
            if connection is None:
                return
            cursor = connection.cursor()
            cursor.execute("SELECT ORCR FROM users WHERE username = %s", (self.logged_in_user['username'],))
            result = cursor.fetchone()
            cursor.close()
            connection.close()

            if result:
                orcr_expiry_date_str = str(result[0])
                orcr_expiry_datetime = datetime.datetime.strptime(orcr_expiry_date_str, "%Y-%m-%d")
                current_datetime = datetime.datetime.now()

                self.ui_welcome.listWidget_3.clear()
                if current_datetime > orcr_expiry_datetime:
                    self.ui_welcome.listWidget_3.addItem("Your OR/CR has expired. Please renew immediately.")
                else:
                    self.ui_welcome.listWidget_3.addItem("Your OR/CR is still up to date.")
        except Exception as e:
            QtWidgets.QMessageBox.critical(self, 'OR/CR Expiry Check Error', f"Error: {e}")

    def check_license_expiry(self):
        try:
            connection = self.db_connect()
            if connection is None:
                return
            cursor = connection.cursor()
            cursor.execute("SELECT LICE FROM users WHERE username = %s", (self.logged_in_user['username'],))
            result = cursor.fetchone()
            cursor.close()
            connection.close()

            if result:
                license_expiry_date_str = str(result[0])
                license_expiry_datetime = datetime.datetime.strptime(license_expiry_date_str, "%Y-%m-%d")
                current_datetime = datetime.datetime.now()

                self.ui_welcome.listWidget_4.clear()
                if current_datetime > license_expiry_datetime:
                    self.ui_welcome.listWidget_4.addItem("Your License has expired. Please renew immediately.")
                else:
                    self.ui_welcome.listWidget_4.addItem("Your License is still up to date.")
        except Exception as e:
            QtWidgets.QMessageBox.critical(self, 'OR/CR Expiry Check Error', f"Error: {e}")
    def db_connect(self):
        try:
            connection = mysql.connector.connect(
                host="localhost",
                user="root",
                password="mysql321!",
                database="user_info"
            )
            return connection
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Connection Error', f"Error: {err}")
            return None

    def login_student(self):
        username = self.ui_input_log.lineEdit_2.text()
        password = self.ui_input_log.lineEdit.text()

        print(f"Attempting login with username: {username}, password: {password}")
        if self.check_student_credentials(username, password):
            user_info = self.get_user_info(username)
            if user_info:
                self.log_login_history(user_info['name'])
            self.welcome()
        else:
            QtWidgets.QMessageBox.warning(self, 'Login Failed', 'Error: Invalid Student Username or Password.')

    def log_login_history(self, name):
        try:
            connection = self.db_connect()
            if connection is None:
                return
            cursor = connection.cursor()
            current_datetime = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            cursor.execute("INSERT INTO login_history (name, login_time) VALUES (%s, %s)",
                           (name, current_datetime))
            connection.commit()
            cursor.close()
            connection.close()
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Error', f"Error: {err}")

    def get_user_info(self, username):
        try:
            connection = self.db_connect()
            if connection is None:
                return None
            cursor = connection.cursor(dictionary=True)
            cursor.execute("SELECT name FROM users WHERE username = %s", (username,))
            user_info = cursor.fetchone()
            cursor.close()
            connection.close()
            return user_info
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Error', f"Error: {err}")
            return None
    def check_student_credentials(self, username, password):
        connection = self.db_connect()
        if connection is None:
            return False
        try:
            cursor = connection.cursor(dictionary=True)
            cursor.execute("SELECT * FROM users WHERE username=%s AND password=%s", (username, password))
            result = cursor.fetchone()
            cursor.close()
            connection.close()
            if result:
                self.logged_in_user = result
                print(f"Login successful: {self.logged_in_user}")
                return True
            print("Login failed: No matching user found")
            return False
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Query Error', f"Error: {err}")
            return False

    def display_user_info(self):
        if self.logged_in_user:
            self.display_orcr_info1()
            self.display_lice_info1()
            self.display_plates_info1()
            self.display_name_info1()

    def display_orcr_info1(self):
        orcr_date = self.logged_in_user.get('ORCR')
        self.ui_welcome.listWidget_6.clear()
        self.ui_welcome.listWidget_6.addItem(f"{orcr_date}")

    def display_lice_info1(self):
        lice_date = self.logged_in_user.get('LICE')
        self.ui_welcome.listWidget_7.clear()
        self.ui_welcome.listWidget_7.addItem(f"{lice_date}")

    def display_plates_info1(self):
        plates = self.logged_in_user.get('PLATES')
        self.ui_welcome.listWidget_8.clear()
        self.ui_welcome.listWidget_8.addItem(f"{plates}")

    def display_name_info1(self):
        name = self.logged_in_user.get('name')
        self.ui_welcome.listWidget.clear()
        self.ui_welcome.listWidget.addItem(f"{name}")

    def admin(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_admins = Ui_MainW2()
        self.ui_admins.setupUi(self.current_window)
        self.current_window.show()
        self.ui_admins.pushButton_2.clicked.connect(self.login_admin)
        self.ui_admins.pushButton_3.clicked.connect(self.input_login)

    def login_admin(self):
        username = self.ui_admins.lineEdit_2.text()
        password = self.ui_admins.lineEdit.text()

        if username == "admin" and password == "123":
            self.admin_display()
        else:
            QtWidgets.QMessageBox.warning(self, 'Login Failed', 'Error Username or Password.')

    def admin_display(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_addis = Ui_Dialog5()
        self.ui_addis.setupUi(self.current_window)
        self.current_window.show()
        self.ui_addis.pushButton1.clicked.connect(self.username)
        self.ui_addis.pushButton_4.clicked.connect(self.platenum)
        self.ui_addis.pushButton_3.clicked.connect(self.input_login)
        self.ui_addis.pushButton_2.clicked.connect(self.ORCR)
        self.ui_addis.pushButton_5.clicked.connect(self.license)
        self.ui_addis.pushButton_6.clicked.connect(self.inbox)
        self.ui_addis.pushButton.clicked.connect(self.history)

    def username(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_yuser = Ui_Dialog6()
        self.ui_yuser.setupUi(self.current_window)
        self.current_window.show()
        self.ui_yuser.pushButton_3.clicked.connect(self.admin_display)
        self.display_credentials()

    def display_credentials(self):
        connection = self.db_connect()
        if connection is None:
            return
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT username, password FROM users")
            result = cursor.fetchall()
            self.ui_yuser.listWidget.clear()
            for row in result:
                username, password = row
                self.ui_yuser.listWidget.addItem(f"USERNAME: {username}, PASSWORD: {password}")
            cursor.close()
            connection.close()
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Query Error', f"Error: {err}")

    def platenum(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_plate = Ui_Dialog7()
        self.ui_plate.setupUi(self.current_window)
        self.current_window.show()
        self.ui_plate.pushButton_3.clicked.connect(self.admin_display)
        self.display_plate_info()

    def display_plate_info(self):
        connection = self.db_connect()
        if connection is None:
            return
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT username, PLATES FROM users")
            result = cursor.fetchall()
            self.ui_plate.listWidget.clear()
            for row in result:
                username, plate_number = row
                self.ui_plate.listWidget.addItem(f"{username}: {plate_number}")
            cursor.close()
            connection.close()
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Query Error', f"Error: {err}")

    def ORCR(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_owr = Ui_Dialog8()
        self.ui_owr.setupUi(self.current_window)
        self.current_window.show()
        self.ui_owr.pushButton_3.clicked.connect(self.admin_display)
        self.display_ocr_info()

    def display_ocr_info(self):
        connection = self.db_connect()
        if connection is None:
            return
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT username, ORCR FROM users")
            result = cursor.fetchall()
            self.ui_owr.listWidget.clear()
            for row in result:
                username, orcr_ex = row
                self.ui_owr.listWidget.addItem(f"{username}: {orcr_ex}")
            cursor.close()
            connection.close()
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Query Error', f"Error: {err}")

    def license(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_lice = Ui_Dialog9()
        self.ui_lice.setupUi(self.current_window)
        self.current_window.show()
        self.ui_lice.pushButton_3.clicked.connect(self.admin_display)
        self.display_lc_info()

    def display_lc_info(self):
        connection = self.db_connect()
        if connection is None:
            return
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT username, LICE FROM users")
            result = cursor.fetchall()
            self.ui_lice.listWidget.clear()
            for row in result:
                username, lice_ex = row
                self.ui_lice.listWidget.addItem(f"{username}: {lice_ex}")
            cursor.close()
            connection.close()
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Query Error', f"Error: {err}")

    def inbox(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_box = Ui_Dialog10()
        self.ui_box.setupUi(self.current_window)
        self.current_window.show()
        self.ui_box.pushButton_3.clicked.connect(self.admin_display)
        self.check_expirations()

    def history(self):
        self.close_current_window()
        self.current_window = QtWidgets.QDialog()
        self.ui_box = Ui_Dialog()
        self.ui_box.setupUi(self.current_window)
        self.current_window.show()
        self.ui_box.pushButton_3.clicked.connect(self.admin_display)
        self.display_login_history()

    def display_login_history(self):
        connection = self.db_connect()
        if connection is None:
            return
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT name, login_time FROM login_history ORDER BY login_time DESC")
            result = cursor.fetchall()
            self.ui_box.listWidget.clear()
            for row in result:
                name, login_time = row
                self.ui_box.listWidget.addItem(f"{name}: {login_time}")
            cursor.close()
            connection.close()
        except mysql.connector.Error as err:
            QtWidgets.QMessageBox.critical(self, 'Database Query Error', f"Error: {err}")
    def check_orcr_expiration(self, orcr_expiry_datetime):
        current_datetime = datetime.datetime.now()
        return current_datetime > orcr_expiry_datetime

    def check_license_expiration(self, license_expiry_datetime):
        current_datetime = datetime.datetime.now()
        return current_datetime > license_expiry_datetime
    def check_expirations(self):
        try:
            connection = self.db_connect()
            if connection is None:
                return
            cursor = connection.cursor(dictionary=True)
            cursor.execute("SELECT username, ORCR, LICE FROM users")
            result = cursor.fetchall()
            cursor.close()
            connection.close()

            self.ui_box.listWidget.clear()
            for row in result:
                username = row['username']
                orcr_expiry_date_str = str(row['ORCR'])
                license_expiry_date_str = str(row['LICE'])
                orcr_expiry_datetime = datetime.datetime.strptime(orcr_expiry_date_str, "%Y-%m-%d")
                license_expiry_datetime = datetime.datetime.strptime(license_expiry_date_str, "%Y-%m-%d")

                orcr_expired = self.check_orcr_expiration(orcr_expiry_datetime)
                license_expired = self.check_license_expiration(license_expiry_datetime)

                if orcr_expired or license_expired:
                    if orcr_expired and license_expired:
                        self.ui_box.listWidget.addItem(f"{username}: The license and the ORCR have expired. Please take action")
                    elif orcr_expired:
                        self.ui_box.listWidget.addItem(f"{username}: The ORCR has expired. Please take action")
                    elif license_expired:
                        self.ui_box.listWidget.addItem(f"{username}: The license has expired. Please take action")
        except Exception as e:
            QtWidgets.QMessageBox.critical(self, 'Expiration Check Error', f"Error: {e}")


if __name__ == "__main__":
    app = QtWidgets.QApplication(sys.argv)
    win = Window()
    win.show()
    sys.exit(app.exec_())
