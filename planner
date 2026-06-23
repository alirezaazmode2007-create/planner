# ai_planner.py
import sys
import json
import os
from datetime import datetime
from PyQt6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QTreeWidget, QTreeWidgetItem, QPushButton, QLabel, QProgressBar,
    QMessageBox, QInputDialog, QLineEdit, QComboBox, QSplitter,
    QFrame, QHeaderView, QTabWidget, QTableWidget, QTableWidgetItem,
    QGroupBox, QGridLayout, QStyleFactory, QScrollArea, QDialog,
    QDialogButtonBox, QFormLayout, QSpinBox, QDoubleSpinBox, QListWidget,
    QListWidgetItem
)
from PyQt6.QtCore import Qt, QPropertyAnimation, QRect
from PyQt6.QtGui import QFont, QPainter, QColor, QPen


class TaskItem:
    """مدل داده برای هر تسک"""

    def __init__(self, phase, activity, status, progress_pct=0):
        self.phase = phase
        self.activity = activity
        self.status = status
        self.progress_pct = progress_pct
        self.created_at = datetime.now().isoformat()

    def to_dict(self):
        return {
            "phase": self.phase,
            "activity": self.activity,
            "status": self.status,
            "progress_pct": self.progress_pct,
            "created_at": self.created_at
        }

    @classmethod
    def from_dict(cls, data):
        task = cls(data["phase"], data["activity"], data["status"], data["progress_pct"])
        task.created_at = data.get("created_at", datetime.now().isoformat())
        return task


class Course:
    """مدل داده برای یک درس"""

    def __init__(self, name, credits, grade=None, semester=None, is_planned=False):
        self.name = name
        self.credits = credits
        self.grade = grade
        self.semester = semester
        self.is_planned = is_planned

    def to_dict(self):
        return {
            "name": self.name,
            "credits": self.credits,
            "grade": self.grade,
            "semester": self.semester,
            "is_planned": self.is_planned
        }

    @classmethod
    def from_dict(cls, data):
        return cls(
            data["name"],
            data["credits"],
            data.get("grade"),
            data.get("semester"),
            data.get("is_planned", False)
        )


class Semester:
    """مدل داده برای یک ترم"""

    def __init__(self, name, year=None):
        self.name = name
        self.year = year or datetime.now().year
        self.courses = []

    def add_course(self, course):
        course.semester = self.name
        self.courses.append(course)

    def get_gpa(self):
        total_credits = 0
        total_points = 0
        for course in self.courses:
            if course.grade is not None and course.grade > 0:
                total_credits += course.credits
                total_points += course.grade * course.credits
        return total_points / total_credits if total_credits > 0 else 0

    def get_total_credits(self):
        return sum(c.credits for c in self.courses)

    def get_completed_credits(self):
        return sum(c.credits for c in self.courses if c.grade is not None and c.grade > 0)

    def to_dict(self):
        return {
            "name": self.name,
            "year": self.year,
            "courses": [c.to_dict() for c in self.courses]
        }

    @classmethod
    def from_dict(cls, data):
        semester = cls(data["name"], data.get("year"))
        for course_data in data.get("courses", []):
            semester.courses.append(Course.from_dict(course_data))
        return semester


class AddCourseDialog(QDialog):
    """دیالوگ افزودن درس جدید"""

    def __init__(self, semester_name, parent=None):
        super().__init__(parent)
        self.semester_name = semester_name
        self.setWindowTitle(f"افزودن درس جدید - {semester_name}")
        self.setModal(True)
        self.setMinimumWidth(400)

        layout = QVBoxLayout(self)

        form_layout = QFormLayout()

        self.name_input = QLineEdit()
        self.name_input.setPlaceholderText("مثال: مبانی برنامه‌نویسی")
        form_layout.addRow("نام درس:", self.name_input)

        self.credits_input = QSpinBox()
        self.credits_input.setRange(1, 6)
        self.credits_input.setValue(3)
        form_layout.addRow("تعداد واحد:", self.credits_input)

        self.grade_input = QDoubleSpinBox()
        self.grade_input.setRange(0, 20)
        self.grade_input.setSingleStep(0.25)
        self.grade_input.setValue(0)
        self.grade_input.setSuffix(" (0 = هنوز گرفته نشده)")
        form_layout.addRow("نمره:", self.grade_input)

        self.is_planned_check = QComboBox()
        self.is_planned_check.addItems(["درس جاری", "درس ترم آینده"])
        form_layout.addRow("وضعیت:", self.is_planned_check)

        layout.addLayout(form_layout)

        buttons = QDialogButtonBox(QDialogButtonBox.StandardButton.Ok | QDialogButtonBox.StandardButton.Cancel)
        buttons.accepted.connect(self.accept)
        buttons.rejected.connect(self.reject)
        layout.addWidget(buttons)

        self.setStyleSheet("""
            QDialog {
                background-color: #2d2d2d;
            }
            QLabel {
                color: #e0e0e0;
            }
            QLineEdit, QSpinBox, QDoubleSpinBox, QComboBox {
                background-color: #3d3d3d;
                color: #e0e0e0;
                border: 1px solid #4d4d4d;
                border-radius: 5px;
                padding: 5px;
            }
            QPushButton {
                background-color: #0d7377;
                color: white;
                border: none;
                border-radius: 5px;
                padding: 8px 20px;
            }
            QPushButton:hover {
                background-color: #0f8a8f;
            }
        """)

    def get_course_data(self):
        return {
            "name": self.name_input.text().strip(),
            "credits": self.credits_input.value(),
            "grade": self.grade_input.value() if self.grade_input.value() > 0 else None,
            "is_planned": self.is_planned_check.currentIndex() == 1
        }


class ProgressCircle(QWidget):
    """ویجت دایره پیشرفت سفارشی"""

    def __init__(self, parent=None):
        super().__init__(parent)
        self.progress = 0
        self.setMinimumSize(80, 80)

    def setProgress(self, value):
        self.progress = value
        self.update()

    def paintEvent(self, event):
        painter = QPainter(self)
        painter.setRenderHint(QPainter.RenderHint.Antialiasing)

        width = self.width()
        height = self.height()
        size = min(width, height) - 10
        rect = QRect(5, 5, size, size)

        painter.setPen(QPen(QColor(60, 60, 60), 6))
        painter.drawArc(rect, 0, 360 * 16)

        painter.setPen(QPen(QColor(0, 200, 100), 6))
        angle = int(360 * 16 * (self.progress / 100))
        painter.drawArc(rect, 90 * 16, -angle)

        painter.setPen(QColor(255, 255, 255))
        font = QFont("Arial", 12, QFont.Weight.Bold)
        painter.setFont(font)
        text = f"{int(self.progress)}%"
        painter.drawText(rect, Qt.AlignmentFlag.AlignCenter, text)


class AnimatedButton(QPushButton):
    """دکمه با افکت انیمیشن"""

    def __init__(self, text, parent=None):
        super().__init__(text, parent)
        self.animation = QPropertyAnimation(self, b"geometry")
        self.animation.setDuration(100)

    def enterEvent(self, event):
        self.animation.stop()
        geo = self.geometry()
        self.animation.setStartValue(geo)
        self.animation.setEndValue(QRect(geo.x() - 2, geo.y() - 2, geo.width() + 4, geo.height() + 4))
        self.animation.start()
        super().enterEvent(event)

    def leaveEvent(self, event):
        self.animation.stop()
        geo = self.geometry()
        self.animation.setStartValue(geo)
        self.animation.setEndValue(QRect(geo.x() + 2, geo.y() + 2, geo.width() - 4, geo.height() - 4))
        self.animation.start()
        super().leaveEvent(event)


class UniversityTab(QWidget):
    """تب مدیریت دانشگاه"""

    def __init__(self, parent=None):
        super().__init__(parent)
        self.parent_app = parent
        self.semesters = []
        self.current_semester_index = -1
        self.data_file = "university_data.json"

        self.setup_ui()
        self.load_data()
        self.refresh_all()

    def setup_ui(self):
        layout = QVBoxLayout(self)
        layout.setSpacing(10)

        header = QLabel("🎓 مدیریت دروس دانشگاهی")
        header.setFont(QFont("Arial", 16, QFont.Weight.Bold))
        header.setStyleSheet("color: #0d7377; padding: 10px;")
        layout.addWidget(header)

        info_frame = QFrame()
        info_frame.setStyleSheet("""
            QFrame {
                background-color: #2d2d2d;
                border-radius: 10px;
                padding: 10px;
            }
        """)
        info_layout = QHBoxLayout(info_frame)

        self.total_credits_label = QLabel("کل واحدهای گذرانده: 0")
        self.total_credits_label.setFont(QFont("Arial", 11))
        info_layout.addWidget(self.total_credits_label)

        self.overall_gpa_label = QLabel("معدل کل: 0.00")
        self.overall_gpa_label.setFont(QFont("Arial", 11, QFont.Weight.Bold))
        self.overall_gpa_label.setStyleSheet("color: #69f0ae;")
        info_layout.addWidget(self.overall_gpa_label)

        info_layout.addStretch()

        add_semester_btn = QPushButton("➕ افزودن ترم جدید")
        add_semester_btn.clicked.connect(self.add_new_semester)
        info_layout.addWidget(add_semester_btn)

        layout.addWidget(info_frame)

        main_splitter = QSplitter(Qt.Orientation.Horizontal)

        left_widget = QWidget()
        left_layout = QVBoxLayout(left_widget)

        semester_label = QLabel("📚 ترم‌های تحصیلی")
        semester_label.setFont(QFont("Arial", 12, QFont.Weight.Bold))
        left_layout.addWidget(semester_label)

        self.semester_list = QListWidget()
        self.semester_list.itemClicked.connect(self.on_semester_selected)
        self.semester_list.setStyleSheet("""
            QListWidget {
                background-color: #252525;
                border: 1px solid #3a3a3a;
                border-radius: 5px;
                padding: 5px;
            }
            QListWidget::item {
                padding: 8px;
                border-radius: 5px;
            }
            QListWidget::item:selected {
                background-color: #0d7377;
            }
            QListWidget::item:hover {
                background-color: #3a3a3a;
            }
        """)
        left_layout.addWidget(self.semester_list)

        semester_actions = QHBoxLayout()
        delete_semester_btn = QPushButton("🗑️ حذف ترم انتخاب شده")
        delete_semester_btn.clicked.connect(self.delete_selected_semester)
        semester_actions.addWidget(delete_semester_btn)
        left_layout.addLayout(semester_actions)

        main_splitter.addWidget(left_widget)

        right_widget = QWidget()
        right_layout = QVBoxLayout(right_widget)

        self.semester_title = QLabel("ترم انتخاب شده: هیچ")
        self.semester_title.setFont(QFont("Arial", 12, QFont.Weight.Bold))
        right_layout.addWidget(self.semester_title)

        self.courses_table = QTableWidget()
        self.courses_table.setColumnCount(5)
        self.courses_table.setHorizontalHeaderLabels(["نام درس", "واحد", "نمره", "وضعیت", "عملیات"])
        self.courses_table.horizontalHeader().setSectionResizeMode(0, QHeaderView.ResizeMode.Stretch)
        self.courses_table.setAlternatingRowColors(True)
        self.courses_table.setStyleSheet("""
            QTableWidget {
                background-color: #252525;
                alternate-background-color: #2a2a2a;
                gridline-color: #3a3a3a;
            }
            QTableWidget::item:selected {
                background-color: #0d7377;
            }
        """)
        right_layout.addWidget(self.courses_table)

        course_actions = QHBoxLayout()

        add_course_btn = QPushButton("➕ افزودن درس")
        add_course_btn.clicked.connect(self.add_course_to_semester)
        course_actions.addWidget(add_course_btn)

        edit_course_btn = QPushButton("✏️ ویرایش نمره")
        edit_course_btn.clicked.connect(self.edit_course_grade)
        course_actions.addWidget(edit_course_btn)

        delete_course_btn = QPushButton("🗑️ حذف درس")
        delete_course_btn.clicked.connect(self.delete_selected_course)
        course_actions.addWidget(delete_course_btn)

        course_actions.addStretch()

        self.semester_gpa_label = QLabel("معدل این ترم: 0.00")
        self.semester_gpa_label.setFont(QFont("Arial", 11, QFont.Weight.Bold))
        self.semester_gpa_label.setStyleSheet("color: #69f0ae;")
        course_actions.addWidget(self.semester_gpa_label)

        right_layout.addLayout(course_actions)

        main_splitter.addWidget(right_widget)
        main_splitter.setSizes([300, 700])

        layout.addWidget(main_splitter)

        plan_frame = QGroupBox("📅 برنامه‌ریزی ترم آینده")
        plan_frame.setFont(QFont("Arial", 11, QFont.Weight.Bold))
        plan_frame.setStyleSheet("""
            QGroupBox {
                background-color: #2d2d2d;
                border: 2px solid #0d7377;
                border-radius: 10px;
                margin-top: 10px;
                padding-top: 15px;
            }
            QGroupBox::title {
                color: #0d7377;
            }
        """)
        plan_layout = QVBoxLayout(plan_frame)

        self.planned_courses_table = QTableWidget()
        self.planned_courses_table.setColumnCount(3)
        self.planned_courses_table.setHorizontalHeaderLabels(["نام درس", "واحد", "عملیات"])
        self.planned_courses_table.horizontalHeader().setSectionResizeMode(0, QHeaderView.ResizeMode.Stretch)
        self.planned_courses_table.setAlternatingRowColors(True)
        plan_layout.addWidget(self.planned_courses_table)

        plan_actions = QHBoxLayout()
        add_planned_btn = QPushButton("➕ افزودن درس برای ترم آینده")
        add_planned_btn.clicked.connect(self.add_planned_course)
        plan_actions.addWidget(add_planned_btn)

        move_to_current_btn = QPushButton("⬆️ انتقال به ترم جاری")
        move_to_current_btn.clicked.connect(self.move_planned_to_current)
        plan_actions.addWidget(move_to_current_btn)

        plan_actions.addStretch()
        plan_layout.addLayout(plan_actions)

        layout.addWidget(plan_frame)

    def add_new_semester(self):
        name, ok = QInputDialog.getText(self, "ترم جدید", "نام ترم را وارد کنید (مثال: ترم ۱, ترم ۲):")
        if ok and name.strip():
            for s in self.semesters:
                if s.name == name.strip():
                    QMessageBox.warning(self, "خطا", "این ترم قبلاً وجود دارد!")
                    return
            semester = Semester(name.strip())
            self.semesters.append(semester)
            self.current_semester_index = len(self.semesters) - 1
            self.save_data()
            self.refresh_all()
            QMessageBox.information(self, "موفق", f"ترم '{name.strip()}' با موفقیت اضافه شد!")

    def delete_selected_semester(self):
        current_item = self.semester_list.currentItem()
        if not current_item:
            QMessageBox.warning(self, "خطا", "لطفاً یک ترم را انتخاب کنید!")
            return

        semester_name = current_item.text().split(" (")[0]
        reply = QMessageBox.question(self, "تأیید حذف",
                                     f"آیا از حذف ترم '{semester_name}' و تمام دروس آن مطمئن هستید؟",
                                     QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)

        if reply == QMessageBox.StandardButton.Yes:
            self.semesters = [s for s in self.semesters if s.name != semester_name]
            self.current_semester_index = -1
            self.save_data()
            self.refresh_all()
            QMessageBox.information(self, "موفق", f"ترم '{semester_name}' با موفقیت حذف شد!")

    def on_semester_selected(self, item):
        semester_name = item.text().split(" (")[0]
        for i, s in enumerate(self.semesters):
            if s.name == semester_name:
                self.current_semester_index = i
                self.refresh_courses_table()
                self.refresh_planned_courses()
                break

    def add_course_to_semester(self):
        if self.current_semester_index == -1:
            QMessageBox.warning(self, "خطا", "لطفاً ابتدا یک ترم را انتخاب کنید!")
            return

        semester = self.semesters[self.current_semester_index]

        dialog = AddCourseDialog(semester.name, self)
        if dialog.exec() == QDialog.DialogCode.Accepted:
            data = dialog.get_course_data()
            if not data["name"]:
                QMessageBox.warning(self, "خطا", "لطفاً نام درس را وارد کنید!")
                return

            for c in semester.courses:
                if c.name == data["name"]:
                    QMessageBox.warning(self, "خطا", "این درس قبلاً در این ترم وجود دارد!")
                    return

            course = Course(data["name"], data["credits"], data["grade"], semester.name, data["is_planned"])

            if data["is_planned"]:
                self.add_to_planned(course)
            else:
                semester.add_course(course)
                self.save_data()
                self.refresh_all()
                QMessageBox.information(self, "موفق", f"درس '{data['name']}' با موفقیت اضافه شد!")

    def add_to_planned(self, course):
        planned_semester = None
        for s in self.semesters:
            if s.name == "ترم آینده":
                planned_semester = s
                break

        if not planned_semester:
            planned_semester = Semester("ترم آینده")
            self.semesters.append(planned_semester)

        for c in planned_semester.courses:
            if c.name == course.name:
                QMessageBox.warning(self, "خطا", "این درس قبلاً در برنامه ترم آینده وجود دارد!")
                return

        planned_semester.add_course(course)
        self.save_data()
        self.refresh_all()
        QMessageBox.information(self, "موفق", f"درس '{course.name}' به برنامه ترم آینده اضافه شد!")

    def edit_course_grade(self):
        if self.current_semester_index == -1:
            QMessageBox.warning(self, "خطا", "لطفاً ابتدا یک ترم را انتخاب کنید!")
            return

        current_row = self.courses_table.currentRow()
        if current_row == -1:
            QMessageBox.warning(self, "خطا", "لطفاً یک درس را انتخاب کنید!")
            return

        semester = self.semesters[self.current_semester_index]
        if current_row >= len(semester.courses):
            return

        course = semester.courses[current_row]

        grade, ok = QInputDialog.getDouble(
            self,
            "ویرایش نمره",
            f"نمره درس '{course.name}' را وارد کنید (0-20):",
            course.grade if course.grade is not None else 0,
            0,
            20,
            2,
            Qt.WindowType.Dialog,
            0.25
        )

        if ok:
            course.grade = grade if grade > 0 else None
            self.save_data()
            self.refresh_courses_table()
            self.update_stats()
            QMessageBox.information(self, "موفق", f"نمره درس '{course.name}' با موفقیت به‌روز شد!")

    def delete_selected_course(self):
        if self.current_semester_index == -1:
            QMessageBox.warning(self, "خطا", "لطفاً ابتدا یک ترم را انتخاب کنید!")
            return

        current_row = self.courses_table.currentRow()
        if current_row == -1:
            QMessageBox.warning(self, "خطا", "لطفاً یک درس را انتخاب کنید!")
            return

        semester = self.semesters[self.current_semester_index]
        if current_row >= len(semester.courses):
            return

        course = semester.courses[current_row]

        reply = QMessageBox.question(self, "تأیید حذف",
                                     f"آیا از حذف درس '{course.name}' مطمئن هستید؟",
                                     QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)

        if reply == QMessageBox.StandardButton.Yes:
            semester.courses.pop(current_row)
            self.save_data()
            self.refresh_courses_table()
            self.update_stats()
            QMessageBox.information(self, "موفق", f"درس '{course.name}' با موفقیت حذف شد!")

    def add_planned_course(self):
        name, ok = QInputDialog.getText(self, "درس جدید", "نام درس را وارد کنید:")
        if ok and name.strip():
            credits, ok = QInputDialog.getInt(self, "تعداد واحد", "تعداد واحد را وارد کنید:", 3, 1, 6)
            if ok:
                course = Course(name.strip(), credits, None, "ترم آینده", True)
                self.add_to_planned(course)

    def move_planned_to_current(self):
        if self.current_semester_index == -1:
            QMessageBox.warning(self, "خطا", "لطفاً ابتدا یک ترم جاری را انتخاب کنید!")
            return

        planned_semester = None
        for s in self.semesters:
            if s.name == "ترم آینده":
                planned_semester = s
                break

        if not planned_semester or not planned_semester.courses:
            QMessageBox.information(self, "اطلاع", "هیچ درسی برای ترم آینده برنامه‌ریزی نشده است!")
            return

        semester = self.semesters[self.current_semester_index]

        for course in planned_semester.courses:
            exists = False
            for c in semester.courses:
                if c.name == course.name:
                    exists = True
                    break
            if not exists:
                new_course = Course(course.name, course.credits, None, semester.name, False)
                semester.courses.append(new_course)

        self.semesters = [s for s in self.semesters if s.name != "ترم آینده"]

        self.save_data()
        self.refresh_all()
        QMessageBox.information(self, "موفق", f"{len(planned_semester.courses)} درس به ترم جاری منتقل شدند!")

    def refresh_all(self):
        self.refresh_semester_list()
        self.refresh_courses_table()
        self.refresh_planned_courses()
        self.update_stats()

    def refresh_semester_list(self):
        self.semester_list.clear()
        for semester in self.semesters:
            if semester.name != "ترم آینده":
                total_credits = semester.get_total_credits()
                gpa = semester.get_gpa()
                item_text = f"{semester.name} (واحد: {total_credits}, معدل: {gpa:.2f})"
                item = QListWidgetItem(item_text)

                if semester.get_completed_credits() == semester.get_total_credits() and semester.get_total_credits() > 0:
                    item.setForeground(QColor(0, 200, 100))
                elif semester.get_completed_credits() > 0:
                    item.setForeground(QColor(255, 200, 0))
                else:
                    item.setForeground(QColor(200, 100, 100))

                self.semester_list.addItem(item)

        if self.semester_list.count() == 0:
            self.semester_list.addItem("هنوز ترمی اضافه نشده است")

    def refresh_courses_table(self):
        self.courses_table.setRowCount(0)

        if self.current_semester_index == -1 or self.current_semester_index >= len(self.semesters):
            self.semester_title.setText("ترم انتخاب شده: هیچ")
            return

        semester = self.semesters[self.current_semester_index]
        self.semester_title.setText(f"ترم انتخاب شده: {semester.name}")

        active_courses = [c for c in semester.courses if not c.is_planned]

        self.courses_table.setRowCount(len(active_courses))

        for row, course in enumerate(active_courses):
            self.courses_table.setItem(row, 0, QTableWidgetItem(course.name))
            self.courses_table.setItem(row, 1, QTableWidgetItem(str(course.credits)))

            grade_text = str(course.grade) if course.grade is not None and course.grade > 0 else "نگرفته"
            grade_item = QTableWidgetItem(grade_text)
            if course.grade is not None and course.grade > 0:
                if course.grade >= 17:
                    grade_item.setForeground(QColor(0, 200, 100))
                elif course.grade >= 12:
                    grade_item.setForeground(QColor(255, 200, 0))
                else:
                    grade_item.setForeground(QColor(200, 100, 100))
            self.courses_table.setItem(row, 2, grade_item)

            status = "✅ گذرانده" if course.grade is not None and course.grade > 0 else "⏳ در حال گذراندن"
            self.courses_table.setItem(row, 3, QTableWidgetItem(status))

            action_btn = QPushButton("✏️ ویرایش")
            action_btn.clicked.connect(lambda checked, r=row: self.edit_course_grade())
            self.courses_table.setCellWidget(row, 4, action_btn)

        gpa = semester.get_gpa()
        self.semester_gpa_label.setText(f"معدل این ترم: {gpa:.2f}")

    def refresh_planned_courses(self):
        self.planned_courses_table.setRowCount(0)

        planned_semester = None
        for s in self.semesters:
            if s.name == "ترم آینده":
                planned_semester = s
                break

        if not planned_semester:
            return

        planned_courses = planned_semester.courses

        self.planned_courses_table.setRowCount(len(planned_courses))

        for row, course in enumerate(planned_courses):
            self.planned_courses_table.setItem(row, 0, QTableWidgetItem(course.name))
            self.planned_courses_table.setItem(row, 1, QTableWidgetItem(str(course.credits)))

            delete_btn = QPushButton("🗑️ حذف")
            delete_btn.clicked.connect(lambda checked, r=row: self.remove_planned_course(r))
            self.planned_courses_table.setCellWidget(row, 2, delete_btn)

    def remove_planned_course(self, row):
        planned_semester = None
        for s in self.semesters:
            if s.name == "ترم آینده":
                planned_semester = s
                break

        if planned_semester and row < len(planned_semester.courses):
            course = planned_semester.courses[row]
            reply = QMessageBox.question(self, "تأیید حذف",
                                         f"آیا از حذف درس برنامه‌ریزی شده '{course.name}' مطمئن هستید؟",
                                         QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)
            if reply == QMessageBox.StandardButton.Yes:
                planned_semester.courses.pop(row)
                if not planned_semester.courses:
                    self.semesters.remove(planned_semester)
                self.save_data()
                self.refresh_all()

    def update_stats(self):
        total_credits = 0
        total_points = 0

        for semester in self.semesters:
            if semester.name != "ترم آینده":
                for course in semester.courses:
                    if not course.is_planned and course.grade is not None and course.grade > 0:
                        total_credits += course.credits
                        total_points += course.grade * course.credits

        self.total_credits_label.setText(f"کل واحدهای گذرانده: {total_credits}")

        overall_gpa = total_points / total_credits if total_credits > 0 else 0
        self.overall_gpa_label.setText(f"معدل کل: {overall_gpa:.2f}")

        if overall_gpa >= 17:
            self.overall_gpa_label.setStyleSheet("color: #69f0ae;")
        elif overall_gpa >= 12:
            self.overall_gpa_label.setStyleSheet("color: #ffd54f;")
        else:
            self.overall_gpa_label.setStyleSheet("color: #ff6b6b;")

    def save_data(self):
        data = [s.to_dict() for s in self.semesters]
        with open(self.data_file, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def load_data(self):
        if os.path.exists(self.data_file):
            try:
                with open(self.data_file, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    self.semesters = [Semester.from_dict(s) for s in data]
            except:
                self.semesters = []
        else:
            self.semesters = []


class AIPlannerApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.tasks = []
        self.phases = {}
        self.data_file = "ai_planner_data.json"

        self.setup_ui()
        self.setup_styles()
        self.load_data()
        self.refresh_all()

    def setup_ui(self):
        self.setWindowTitle("AI Learning Roadmap Planner - Professional Tracker")
        self.setGeometry(100, 100, 1400, 850)

        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QVBoxLayout(central_widget)

        header = QFrame()
        header.setFixedHeight(100)
        header.setObjectName("header")
        header_layout = QHBoxLayout(header)

        title = QLabel("🎯 AI & Data Engineering Learning Roadmap")
        title.setFont(QFont("Arial", 20, QFont.Weight.Bold))
        title.setStyleSheet("color: white;")

        subtitle = QLabel("Professional Progress Tracker + University Manager")
        subtitle.setFont(QFont("Arial", 12))
        subtitle.setStyleSheet("color: #cccccc;")

        header_text_layout = QVBoxLayout()
        header_text_layout.addWidget(title)
        header_text_layout.addWidget(subtitle)

        self.overall_progress_label = QLabel("پیشرفت کلی: 0%")
        self.overall_progress_label.setFont(QFont("Arial", 14, QFont.Weight.Bold))
        self.overall_progress_label.setStyleSheet(
            "color: white; background-color: #2d2d2d; padding: 10px; border-radius: 10px;")

        header_layout.addLayout(header_text_layout)
        header_layout.addStretch()
        header_layout.addWidget(self.overall_progress_label)

        main_layout.addWidget(header)

        self.tab_widget = QTabWidget()
        self.tab_widget.setFont(QFont("Arial", 11))

        roadmap_tab = QWidget()
        roadmap_layout = QHBoxLayout(roadmap_tab)

        splitter = QSplitter(Qt.Orientation.Horizontal)

        sidebar = self.create_sidebar()
        splitter.addWidget(sidebar)

        self.tree_widget = QTreeWidget()
        self.tree_widget.setHeaderLabels(["فاز", "فعالیت", "وضعیت", "پیشرفت"])
        self.tree_widget.setAlternatingRowColors(True)
        self.tree_widget.setFont(QFont("Arial", 10))
        self.tree_widget.setIndentation(20)

        header_widths = [200, 450, 120, 100]
        for i, width in enumerate(header_widths):
            self.tree_widget.header().setSectionResizeMode(i, QHeaderView.ResizeMode.Interactive)
            self.tree_widget.setColumnWidth(i, width)

        self.tree_widget.itemDoubleClicked.connect(self.on_task_double_clicked)
        splitter.addWidget(self.tree_widget)

        splitter.setSizes([300, 1100])
        roadmap_layout.addWidget(splitter)

        self.tab_widget.addTab(roadmap_tab, "📋 نقشه راه")

        dashboard_tab = self.create_dashboard_tab()
        self.tab_widget.addTab(dashboard_tab, "📊 داشبورد")

        self.university_tab = UniversityTab(self)
        self.tab_widget.addTab(self.university_tab, "🎓 دانشگاه")

        add_task_tab = self.create_add_task_tab()
        self.tab_widget.addTab(add_task_tab, "➕ افزودن تسک جدید")

        stats_tab = self.create_stats_tab()
        self.tab_widget.addTab(stats_tab, "📈 آمار و گزارش")

        main_layout.addWidget(self.tab_widget)

        self.statusBar().showMessage("آماده - برای تغییر وضعیت روی آیتم دابل کلیک کنید")
        self.statusBar().setFont(QFont("Arial", 9))

    def create_sidebar(self):
        sidebar = QWidget()
        sidebar.setFixedWidth(280)
        sidebar.setObjectName("sidebar")
        layout = QVBoxLayout(sidebar)
        layout.setSpacing(15)

        filter_group = QGroupBox("🔍 فیلتر بر اساس فاز")
        filter_group.setFont(QFont("Arial", 10, QFont.Weight.Bold))
        filter_layout = QVBoxLayout(filter_group)

        self.filter_combo = QComboBox()
        self.filter_combo.addItem("همه فازها")
        self.filter_combo.currentTextChanged.connect(self.filter_tasks)
        filter_layout.addWidget(self.filter_combo)

        layout.addWidget(filter_group)

        status_group = QGroupBox("🎯 فیلتر وضعیت")
        status_layout = QVBoxLayout(status_group)

        self.status_filter_combo = QComboBox()
        self.status_filter_combo.addItems(["همه", "Done", "In Progress", "Not Done"])
        self.status_filter_combo.currentTextChanged.connect(self.filter_tasks)
        status_layout.addWidget(self.status_filter_combo)

        layout.addWidget(status_group)

        quick_actions = QGroupBox("⚡ اقدامات سریع")
        quick_layout = QVBoxLayout(quick_actions)

        mark_all_done_btn = AnimatedButton("✅ علامت‌گذاری همه به عنوان Done")
        mark_all_done_btn.clicked.connect(self.mark_all_done)
        quick_layout.addWidget(mark_all_done_btn)

        reset_all_btn = AnimatedButton("🔄 ریست همه تسک‌ها")
        reset_all_btn.clicked.connect(self.reset_all_tasks)
        quick_layout.addWidget(reset_all_btn)

        export_btn = AnimatedButton("💾 خروجی JSON")
        export_btn.clicked.connect(self.export_data)
        quick_layout.addWidget(export_btn)

        layout.addWidget(quick_actions)

        progress_group = QGroupBox("📊 پیشرفت فازها")
        progress_group.setFont(QFont("Arial", 10, QFont.Weight.Bold))
        progress_layout = QVBoxLayout(progress_group)

        self.phase_progress_widget = QWidget()
        self.phase_progress_layout = QVBoxLayout(self.phase_progress_widget)
        self.phase_progress_layout.setSpacing(8)

        scroll = QScrollArea()
        scroll.setWidget(self.phase_progress_widget)
        scroll.setWidgetResizable(True)
        scroll.setMaximumHeight(400)
        progress_layout.addWidget(scroll)

        layout.addWidget(progress_group)
        layout.addStretch()

        return sidebar

    def create_dashboard_tab(self):
        tab = QWidget()
        layout = QGridLayout(tab)
        layout.setSpacing(20)

        self.progress_circles = {}
        phases = ["فاز ۰", "فاز ۱", "فاز ۲", "فاز ۳", "فاز ۴", "فاز ۵", "فاز ۶", "فاز ۷"]
        phase_names = ["فونداسیون", "مهندسی داده", "ریاضیات ML", "ML کلاسیک", "MLOps", "LLM و RAG", "AI Agents",
                       "بازار کار"]

        for i, (phase, name) in enumerate(zip(phases, phase_names)):
            col = i % 4
            row = i // 4

            group = QGroupBox(f"{phase}: {name}")
            group_layout = QVBoxLayout(group)

            circle = ProgressCircle()
            circle.setFixedSize(100, 100)
            group_layout.addWidget(circle, alignment=Qt.AlignmentFlag.AlignCenter)

            label = QLabel("0%")
            label.setAlignment(Qt.AlignmentFlag.AlignCenter)
            label.setFont(QFont("Arial", 12, QFont.Weight.Bold))
            group_layout.addWidget(label)

            layout.addWidget(group, row, col)
            self.progress_circles[f"{phase}: {name}"] = (circle, label)

        overall_group = QGroupBox("پیشرفت کلی دوره")
        overall_layout = QVBoxLayout(overall_group)

        self.overall_bar = QProgressBar()
        self.overall_bar.setFont(QFont("Arial", 12))
        self.overall_bar.setFixedHeight(30)
        self.overall_bar.setStyleSheet("""
            QProgressBar {
                border: 2px solid #3a3a3a;
                border-radius: 15px;
                text-align: center;
                background-color: #2d2d2d;
            }
            QProgressBar::chunk {
                background-color: qlineargradient(x1:0, y1:0, x2:1, y2:0,
                    stop:0 #00c853, stop:1 #69f0ae);
                border-radius: 13px;
            }
        """)
        overall_layout.addWidget(self.overall_bar)

        layout.addWidget(overall_group, 2, 0, 1, 4)

        return tab

    def create_add_task_tab(self):
        tab = QWidget()
        layout = QVBoxLayout(tab)
        layout.setSpacing(15)

        form_group = QGroupBox("➕ افزودن فعالیت جدید")
        form_group.setFont(QFont("Arial", 12, QFont.Weight.Bold))
        form_layout = QGridLayout(form_group)

        form_layout.addWidget(QLabel("فاز:"), 0, 0)
        self.new_phase_combo = QComboBox()
        self.new_phase_combo.setEditable(True)
        self.new_phase_combo.addItems(["فاز ۰: فونداسیون", "فاز ۱: مهندسی داده", "فاز ۲: ریاضیات ML",
                                       "فاز ۳: ML کلاسیک", "فاز ۴: MLOps", "فاز ۵: LLM و RAG",
                                       "فاز ۶: AI Agents", "فاز ۷: بازار کار"])
        form_layout.addWidget(self.new_phase_combo, 0, 1)

        form_layout.addWidget(QLabel("عنوان فعالیت:"), 1, 0)
        self.new_activity_input = QLineEdit()
        self.new_activity_input.setPlaceholderText("مثال: یادگیری FastAPI پیشرفته")
        form_layout.addWidget(self.new_activity_input, 1, 1)

        form_layout.addWidget(QLabel("وضعیت اولیه:"), 2, 0)
        self.new_status_combo = QComboBox()
        self.new_status_combo.addItems(["Not Done", "In Progress", "Done"])
        form_layout.addWidget(self.new_status_combo, 2, 1)

        add_btn = AnimatedButton("➕ افزودن تسک جدید")
        add_btn.clicked.connect(self.add_new_task)
        form_layout.addWidget(add_btn, 3, 0, 1, 2)

        layout.addWidget(form_group)

        recent_group = QGroupBox("📝 تسک‌های اضافه شده توسط شما")
        recent_layout = QVBoxLayout(recent_group)

        self.custom_tasks_table = QTableWidget()
        self.custom_tasks_table.setColumnCount(3)
        self.custom_tasks_table.setHorizontalHeaderLabels(["فاز", "فعالیت", "وضعیت"])
        self.custom_tasks_table.horizontalHeader().setSectionResizeMode(1, QHeaderView.ResizeMode.Stretch)
        recent_layout.addWidget(self.custom_tasks_table)

        layout.addWidget(recent_group)

        return tab

    def create_stats_tab(self):
        tab = QWidget()
        layout = QVBoxLayout(tab)

        stats_group = QGroupBox("📊 آمار کلی پیشرفت")
        stats_layout = QGridLayout(stats_group)

        self.total_tasks_label = QLabel("کل تسک‌ها: 0")
        self.total_tasks_label.setFont(QFont("Arial", 12))
        stats_layout.addWidget(self.total_tasks_label, 0, 0)

        self.done_tasks_label = QLabel("تسک‌های انجام شده: 0")
        self.done_tasks_label.setFont(QFont("Arial", 12))
        stats_layout.addWidget(self.done_tasks_label, 0, 1)

        self.inprogress_tasks_label = QLabel("تسک‌های در حال انجام: 0")
        self.inprogress_tasks_label.setFont(QFont("Arial", 12))
        stats_layout.addWidget(self.inprogress_tasks_label, 1, 0)

        self.notdone_tasks_label = QLabel("تسک‌های انجام نشده: 0")
        self.notdone_tasks_label.setFont(QFont("Arial", 12))
        stats_layout.addWidget(self.notdone_tasks_label, 1, 1)

        layout.addWidget(stats_group)

        phase_group = QGroupBox("📋 خلاصه پیشرفت فازها")
        phase_layout = QVBoxLayout(phase_group)

        self.phase_summary_table = QTableWidget()
        self.phase_summary_table.setColumnCount(4)
        self.phase_summary_table.setHorizontalHeaderLabels(["فاز", "کل تسک‌ها", "تکمیل شده", "درصد"])
        self.phase_summary_table.horizontalHeader().setSectionResizeMode(0, QHeaderView.ResizeMode.ResizeToContents)
        phase_layout.addWidget(self.phase_summary_table)

        layout.addWidget(phase_group)

        return tab

    def setup_styles(self):
        self.setStyleSheet("""
            QMainWindow {
                background-color: #1e1e1e;
            }
            QWidget {
                background-color: #1e1e1e;
                color: #e0e0e0;
                font-family: 'Arial';
            }
            QHeaderView::section {
                background-color: #2d2d2d;
                padding: 8px;
                border: 1px solid #3a3a3a;
                font-weight: bold;
            }
            QTreeWidget {
                background-color: #252525;
                alternate-background-color: #2a2a2a;
                border: 1px solid #3a3a3a;
                outline: none;
            }
            QTreeWidget::item {
                padding: 5px;
            }
            QTreeWidget::item:selected {
                background-color: #0d7377;
            }
            QGroupBox {
                font-weight: bold;
                border: 2px solid #3a3a3a;
                border-radius: 8px;
                margin-top: 10px;
                padding-top: 10px;
            }
            QGroupBox::title {
                subcontrol-origin: margin;
                left: 10px;
                padding: 0 5px 0 5px;
            }
            QComboBox {
                background-color: #2d2d2d;
                border: 1px solid #3a3a3a;
                border-radius: 5px;
                padding: 6px;
            }
            QComboBox:hover {
                border-color: #0d7377;
            }
            QComboBox::drop-down {
                border: none;
            }
            QLineEdit {
                background-color: #2d2d2d;
                border: 1px solid #3a3a3a;
                border-radius: 5px;
                padding: 6px;
            }
            QLineEdit:focus {
                border-color: #0d7377;
            }
            QPushButton {
                background-color: qlineargradient(x1:0, y1:0, x2:0, y2:1,
                    stop:0 #0d7377, stop:1 #0a5a5e);
                border: none;
                border-radius: 6px;
                padding: 8px;
                font-weight: bold;
            }
            QPushButton:hover {
                background-color: qlineargradient(x1:0, y1:0, x2:0, y2:1,
                    stop:0 #0f8a8f, stop:1 #0d7377);
            }
            QPushButton:pressed {
                background-color: #0a5a5e;
            }
            QTabWidget::pane {
                border: 1px solid #3a3a3a;
                border-radius: 5px;
            }
            QTabBar::tab {
                background-color: #2d2d2d;
                padding: 10px 20px;
                margin-right: 2px;
                border-top-left-radius: 5px;
                border-top-right-radius: 5px;
            }
            QTabBar::tab:selected {
                background-color: #0d7377;
            }
            QTabBar::tab:hover:!selected {
                background-color: #3a3a3a;
            }
            QTableWidget {
                background-color: #252525;
                alternate-background-color: #2a2a2a;
                gridline-color: #3a3a3a;
            }
            QTableWidget::item:selected {
                background-color: #0d7377;
            }
            QScrollArea {
                border: none;
                background-color: transparent;
            }
        """)

        header = self.findChild(QFrame, "header")
        if header:
            header.setStyleSheet("""
                QFrame {
                    background-color: qlineargradient(x1:0, y1:0, x2:1, y2:0,
                        stop:0 #0d7377, stop:1 #143d42);
                    border-radius: 10px;
                }
            """)

    def load_data(self):
        if os.path.exists(self.data_file):
            try:
                with open(self.data_file, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    self.tasks = [TaskItem.from_dict(t) for t in data]
            except:
                self.tasks = []
        else:
            self.tasks = []

        # اگر هیچ تسکی وجود نداشت، چند نمونه برای شروع اضافه می‌کنیم
        if not self.tasks:
            sample_tasks = [
                ("فاز ۰: فونداسیون", "نمونه تسک ۱ - این یک تسک نمونه است", "Not Done"),
                ("فاز ۰: فونداسیون", "نمونه تسک ۲ - این یک تسک نمونه دیگر است", "Not Done"),
                ("فاز ۱: مهندسی داده", "نمونه تسک ۳ - در این فاز قرار دارد", "In Progress"),
            ]
            self.tasks = [TaskItem(p, a, s) for p, a, s in sample_tasks]

    def save_data(self):
        data = [t.to_dict() for t in self.tasks]
        with open(self.data_file, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def organize_phases(self):
        self.phases = {}
        for task in self.tasks:
            if task.phase not in self.phases:
                self.phases[task.phase] = []
            self.phases[task.phase].append(task)

        phases_list = list(self.phases.keys())
        self.filter_combo.clear()
        self.filter_combo.addItem("همه فازها")
        self.filter_combo.addItems(phases_list)

    def refresh_all(self):
        self.organize_phases()
        self.refresh_tree()
        self.refresh_sidebar_progress()
        self.refresh_dashboard()
        self.refresh_stats_tab()
        self.refresh_custom_tasks_table()
        self.save_data()

    def refresh_tree(self):
        self.tree_widget.clear()

        current_filter = self.filter_combo.currentText()
        status_filter = self.status_filter_combo.currentText()

        for phase, tasks in self.phases.items():
            if current_filter != "همه فازها" and current_filter != phase:
                continue

            phase_item = QTreeWidgetItem([phase, "", "", ""])
            phase_item.setFont(0, QFont("Arial", 11, QFont.Weight.Bold))
            phase_item.setForeground(0, QColor(0, 200, 150))

            phase_total = len(tasks)
            phase_done = sum(1 for t in tasks if t.status == "Done")
            phase_progress = int((phase_done / phase_total) * 100) if phase_total > 0 else 0

            phase_item.setText(3, f"{phase_progress}%")

            for task in tasks:
                if status_filter != "همه" and task.status != status_filter:
                    continue

                task_item = QTreeWidgetItem(["", task.activity, task.status, ""])

                if task.status == "Done":
                    task_item.setForeground(2, QColor(0, 200, 100))
                elif task.status == "In Progress":
                    task_item.setForeground(2, QColor(255, 200, 0))
                else:
                    task_item.setForeground(2, QColor(200, 100, 100))

                progress = 100 if task.status == "Done" else (50 if task.status == "In Progress" else 0)
                task_item.setText(3, f"{progress}%")

                phase_item.addChild(task_item)

            if phase_item.childCount() > 0:
                self.tree_widget.addTopLevelItem(phase_item)
                phase_item.setExpanded(True)

    def refresh_sidebar_progress(self):
        for i in reversed(range(self.phase_progress_layout.count())):
            widget = self.phase_progress_layout.itemAt(i).widget()
            if widget:
                widget.deleteLater()

        for phase, tasks in self.phases.items():
            phase_total = len(tasks)
            phase_done = sum(1 for t in tasks if t.status == "Done")
            phase_progress = int((phase_done / phase_total) * 100) if phase_total > 0 else 0

            progress_frame = QWidget()
            progress_layout = QVBoxLayout(progress_frame)
            progress_layout.setSpacing(3)

            phase_label = QLabel(phase)
            phase_label.setFont(QFont("Arial", 9))

            progress_bar = QProgressBar()
            progress_bar.setValue(phase_progress)
            progress_bar.setFixedHeight(12)
            progress_bar.setTextVisible(False)
            progress_bar.setStyleSheet("""
                QProgressBar {
                    border: none;
                    border-radius: 6px;
                    background-color: #3a3a3a;
                }
                QProgressBar::chunk {
                    background-color: #0d7377;
                    border-radius: 6px;
                }
            """)

            percent_label = QLabel(f"{phase_progress}%")
            percent_label.setAlignment(Qt.AlignmentFlag.AlignRight)
            percent_label.setFont(QFont("Arial", 8))

            header_layout = QHBoxLayout()
            header_layout.addWidget(phase_label)
            header_layout.addStretch()
            header_layout.addWidget(percent_label)

            progress_layout.addLayout(header_layout)
            progress_layout.addWidget(progress_bar)

            self.phase_progress_layout.addWidget(progress_frame)

        total_tasks = len(self.tasks)
        total_done = sum(1 for t in self.tasks if t.status == "Done")
        overall_progress = int((total_done / total_tasks) * 100) if total_tasks > 0 else 0
        self.overall_progress_label.setText(f"پیشرفت کلی: {overall_progress}%")

    def refresh_dashboard(self):
        for phase, tasks in self.phases.items():
            phase_done = sum(1 for t in tasks if t.status == "Done")
            phase_total = len(tasks)
            progress = int((phase_done / phase_total) * 100) if phase_total > 0 else 0

            for circle_name, (circle, label) in self.progress_circles.items():
                if circle_name.startswith(phase.split(":")[0]):
                    circle.setProgress(progress)
                    label.setText(f"{progress}%")
                    break

        total_tasks = len(self.tasks)
        total_done = sum(1 for t in self.tasks if t.status == "Done")
        overall_progress = int((total_done / total_tasks) * 100) if total_tasks > 0 else 0
        self.overall_bar.setValue(overall_progress)

    def refresh_stats_tab(self):
        total_tasks = len(self.tasks)
        total_done = sum(1 for t in self.tasks if t.status == "Done")
        total_inprogress = sum(1 for t in self.tasks if t.status == "In Progress")
        total_notdone = sum(1 for t in self.tasks if t.status == "Not Done")

        self.total_tasks_label.setText(f"کل تسک‌ها: {total_tasks}")
        self.done_tasks_label.setText(f"تسک‌های انجام شده: {total_done}")
        self.inprogress_tasks_label.setText(f"تسک‌های در حال انجام: {total_inprogress}")
        self.notdone_tasks_label.setText(f"تسک‌های انجام نشده: {total_notdone}")

        self.phase_summary_table.setRowCount(len(self.phases))
        for row, (phase, tasks) in enumerate(self.phases.items()):
            phase_total = len(tasks)
            phase_done = sum(1 for t in tasks if t.status == "Done")
            phase_progress = int((phase_done / phase_total) * 100) if phase_total > 0 else 0

            self.phase_summary_table.setItem(row, 0, QTableWidgetItem(phase))
            self.phase_summary_table.setItem(row, 1, QTableWidgetItem(str(phase_total)))
            self.phase_summary_table.setItem(row, 2, QTableWidgetItem(str(phase_done)))

            progress_item = QTableWidgetItem(f"{phase_progress}%")
            if phase_progress == 100:
                progress_item.setForeground(QColor(0, 200, 100))
            elif phase_progress >= 50:
                progress_item.setForeground(QColor(255, 200, 0))
            else:
                progress_item.setForeground(QColor(200, 100, 100))
            self.phase_summary_table.setItem(row, 3, progress_item)

        self.phase_summary_table.resizeColumnsToContents()

    def refresh_custom_tasks_table(self):
        sample_activities = {"نمونه تسک ۱ - این یک تسک نمونه است", "نمونه تسک ۲ - این یک تسک نمونه دیگر است", "نمونه تسک ۳ - در این فاز قرار دارد"}
        custom_tasks = [t for t in self.tasks if t.activity not in sample_activities]

        self.custom_tasks_table.setRowCount(len(custom_tasks))
        for row, task in enumerate(custom_tasks):
            self.custom_tasks_table.setItem(row, 0, QTableWidgetItem(task.phase))
            self.custom_tasks_table.setItem(row, 1, QTableWidgetItem(task.activity))

            status_item = QTableWidgetItem(task.status)
            if task.status == "Done":
                status_item.setForeground(QColor(0, 200, 100))
            elif task.status == "In Progress":
                status_item.setForeground(QColor(255, 200, 0))
            else:
                status_item.setForeground(QColor(200, 100, 100))
            self.custom_tasks_table.setItem(row, 2, status_item)

        self.custom_tasks_table.resizeColumnsToContents()

    def on_task_double_clicked(self, item, column):
        if item.parent() is None:
            return

        phase_item = item.parent()
        phase_name = phase_item.text(0)
        activity_name = item.text(1)

        for task in self.tasks:
            if task.phase == phase_name and task.activity == activity_name:
                if task.status == "Not Done":
                    task.status = "In Progress"
                elif task.status == "In Progress":
                    task.status = "Done"
                else:
                    task.status = "Not Done"
                break

        self.refresh_all()
        self.statusBar().showMessage(f"وضعیت '{activity_name}' تغییر کرد", 2000)

    def filter_tasks(self):
        self.refresh_tree()

    def mark_all_done(self):
        reply = QMessageBox.question(self, "تأیید", "آیا از علامت‌گذاری همه تسک‌ها به عنوان انجام شده مطمئن هستید؟",
                                     QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)
        if reply == QMessageBox.StandardButton.Yes:
            for task in self.tasks:
                task.status = "Done"
            self.refresh_all()
            self.statusBar().showMessage("همه تسک‌ها به عنوان انجام شده علامت‌گذاری شدند", 3000)

    def reset_all_tasks(self):
        reply = QMessageBox.question(self, "تأیید", "آیا از ریست کردن همه تسک‌ها مطمئن هستید؟",
                                     QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)
        if reply == QMessageBox.StandardButton.Yes:
            for task in self.tasks:
                task.status = "Not Done"
            self.refresh_all()
            self.statusBar().showMessage("همه تسک‌ها ریست شدند", 3000)

    def export_data(self):
        export_file = f"ai_planner_export_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"

        with open(export_file, 'w', encoding='utf-8') as f:
            json.dump([t.to_dict() for t in self.tasks], f, ensure_ascii=False, indent=2)

        QMessageBox.information(self, "خروجی گرفته شد", f"داده‌ها با موفقیت در فایل {export_file} ذخیره شدند")

    def add_new_task(self):
        phase = self.new_phase_combo.currentText()
        activity = self.new_activity_input.text().strip()
        status = self.new_status_combo.currentText()

        if not activity:
            QMessageBox.warning(self, "خطا", "لطفاً عنوان فعالیت را وارد کنید!")
            return

        for task in self.tasks:
            if task.phase == phase and task.activity == activity:
                QMessageBox.warning(self, "خطا", "این فعالیت قبلاً در این فاز وجود دارد!")
                return

        new_task = TaskItem(phase, activity, status)
        self.tasks.append(new_task)

        self.new_activity_input.clear()
        self.new_status_combo.setCurrentIndex(0)

        self.refresh_all()
        self.tab_widget.setCurrentIndex(0)
        self.statusBar().showMessage(f"تسک جدید '{activity}' با موفقیت اضافه شد", 3000)


def main():
    app = QApplication(sys.argv)
    app.setStyle(QStyleFactory.create("Fusion"))

    window = AIPlannerApp()
    window.show()

    sys.exit(app.exec())


if __name__ == "__main__":
    main()
