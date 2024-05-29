import sys
import numpy as np
from PySide6.QtCore import QSize, Qt, QPoint
from PySide6.QtGui import QAction, QIcon, QKeySequence, QImage, QPixmap, QWheelEvent
from PySide6.QtWidgets import (
    QApplication,
    QLabel,
    QMainWindow,
    QScrollArea,
    QSizePolicy,
    QFileDialog,
    QGridLayout,
    QWidget,
)
from osgeo import gdal

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()

        self.setWindowTitle("Our App")
        self.base_image_path = None
        self.drone_image_path = None
        self.setGeometry(100, 100, 1200, 600)  # 调整窗口大小以适应两个图像视图

        self.base_image_label = QLabel()  # 用于显示基准影像的Qlabel
        self.drone_image_label = QLabel()
        # 自动适应图像的大小
        self.base_image_label.setScaledContents(True)
        self.drone_image_label.setScaledContents(True)

        # 使用QScrollArea类创建一个滚动区域对象，并将其命名为self.base_scroll_area
        self.base_scroll_area = QScrollArea(self)
        self.base_scroll_area.setWidget(self.base_image_label)
        self.drone_scroll_area = QScrollArea(self)
        self.drone_scroll_area.setWidget(self.drone_image_label)

        # 创建一个打开的菜单
        menu = self.menuBar()
        file_menu = menu.addMenu("打开")
        conduct_menu = menu.addMenu("处理")
        file_menu.addSeparator()
        file_submenu1 = file_menu.addMenu("打开基准影像")
        file_menu.addSeparator()
        file_submenu2 = file_menu.addMenu("打开待配准的无人机影像")

        conduct_menu.addSeparator()
        conduct_menu1 = conduct_menu.addMenu("预处理")
        conduct_menu.addSeparator()
        conduct_menu1 = conduct_menu.addMenu("   ")

        # 为菜单连接槽函数
        button_action1 = QAction("tiff、png、jpg", self)
        button_action1.triggered.connect(self.open_base_image)
        file_submenu1.addAction(button_action1)

        button_action2 = QAction("tiff、png、JPG", self)
        button_action2.triggered.connect(self.open_drone_image)
        file_submenu2.addAction(button_action2)

        button_action_yuchuli = QAction("  ", self)
        button_action_yuchuli.triggered.connect(self.yuchuli)

        # 创建主窗口布局
        main_layout = QGridLayout()

        # 将 QLabel 放置在 QScrollArea 中
        main_layout.addWidget(self.base_scroll_area, 0, 0)  # 基准影像在左侧
        main_layout.addWidget(self.drone_scroll_area, 0, 1)  # 无人机影像在右侧

        self.base_image_label.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Expanding)
        self.drone_image_label.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Expanding)

        central_widget = QWidget()
        central_widget.setLayout(main_layout)
        self.setCentralWidget(central_widget)

        self.m_scaleValue = 1.0  # 初始缩放比例

    def open_base_image(self):
        file_path, _ = QFileDialog.getOpenFileName(
            self, "Open Image", "", "Image Files (*.png *.jpg *.tif *.tiff);;All Files (*)"
        )
        if file_path:
            self.base_image_path = file_path
            if file_path.endswith((".tif", ".tiff")):
                dataset = gdal.Open(self.base_image_path)
                width = dataset.RasterXSize
                height = dataset.RasterYSize
                bands = dataset.RasterCount
                image_data = dataset.ReadAsArray().reshape(height, width, bands)
                if bands == 1:
                    image_data = image_data.astype(np.uint8)
                if bands == 1:
                    image = QImage(image_data.flatten(), width, height, QImage.Format_Grayscale8)
                elif bands == 3:
                    image = QImage(image_data.tobytes(), width, height, QImage.Format_RGB888)
                else:
                    print("不支持的波段数")
                    return
            else:
                image = QImage(self.base_image_path)
            pixmap = QPixmap.fromImage(image)
            self.base_image_label.setPixmap(pixmap)
            self.base_image_label.adjustSize()
            self.adjust_image_width_to_view(self.base_image_label, self.base_scroll_area)

            # 计算并设置初始滚动位置
            self.center_image_in_scroll_area(self.base_image_label, self.base_scroll_area)

    def open_drone_image(self):
        file_path, _ = QFileDialog.getOpenFileName(
            self, "Open Image", "", "Image Files (*.png *.jpg *.tif *.tiff);;All Files (*)"
        )
        if file_path:
            self.drone_image_path = file_path
            if file_path.endswith((".tif", ".tiff")):
                dataset = gdal.Open(self.drone_image_path)
                width = dataset.RasterXSize
                height = dataset.RasterYSize
                bands = dataset.RasterCount
                image_data = dataset.ReadAsArray().reshape(height, width, bands)
                if bands == 1:
                    image_data = image_data.astype(np.uint8)
                if bands == 1:
                    image = QImage(image_data.flatten(), width, height, QImage.Format_Grayscale8)
                elif bands == 3:
                    image = QImage(image_data.tobytes(), width, height, QImage.Format_RGB888)
                else:
                    print("不支持的波段数")
                    return
            else:
                image = QImage(self.drone_image_path)
            pixmap = QPixmap.fromImage(image)
            self.drone_image_label.setPixmap(pixmap)
            self.drone_image_label.adjustSize()
            self.adjust_image_width_to_view(self.drone_image_label, self.drone_scroll_area)

            # 计算并设置初始滚动位置
            self.center_image_in_scroll_area(self.drone_image_label, self.drone_scroll_area)

    def yuchuli(self):
        print("open")

    def wheelEvent(self, event):
        self.zoom(event)

    def zoom(self, event):
        old_scale_value = self.m_scaleValue
        delta = event.angleDelta().y()
        if delta > 0:
            factor = 1.1  # 放大因子
        else:
            factor = 0.9  # 缩小因子
        self.m_scaleValue *= factor
        if not (0.01 < self.m_scaleValue < 1000):
            self.m_scaleValue = old_scale_value
            return

        # 获取鼠标相对于窗口的位置
        mouse_pos = event.position().toPoint()
        # 获取鼠标相对于图像的位置
        base_mouse_pos = self.base_image_label.mapFrom(self, mouse_pos)
        drone_mouse_pos = self.drone_image_label.mapFrom(self, mouse_pos)

        # 缩放图像
        self.base_image_label.resize(self.m_scaleValue * self.base_image_label.pixmap().size())
        self.drone_image_label.resize(self.m_scaleValue * self.drone_image_label.pixmap().size())

        # 计算新的滚动条位置以保持鼠标位置在缩放中心
        new_base_pos = (base_mouse_pos * factor) - mouse_pos
        new_drone_pos = (drone_mouse_pos * factor) - mouse_pos

        self.base_scroll_area.horizontalScrollBar().setValue(int(new_base_pos.x()))
        self.base_scroll_area.verticalScrollBar().setValue(int(new_base_pos.y()))

        self.drone_scroll_area.horizontalScrollBar().setValue(int(new_drone_pos.x()))
        self.drone_scroll_area.verticalScrollBar().setValue(int(new_drone_pos.y()))

    def adjust_image_width_to_view(self, label, scroll_area):
        pixmap = label.pixmap()
        if pixmap:
            scroll_area_width = scroll_area.viewport().width()
            pixmap_width = pixmap.width()
            scale_factor = scroll_area_width / pixmap_width
            new_size = pixmap.size() * scale_factor
            label.resize(new_size)
            self.m_scaleValue = scale_factor

    def center_image_in_scroll_area(self, label, scroll_area):
        pixmap = label.pixmap()
        if pixmap:
            scroll_area_width = scroll_area.viewport().width()
            scroll_area_height = scroll_area.viewport().height()
            pixmap_width = pixmap.width()
            pixmap_height = pixmap.height()

            # 计算水平滚动位置
            horizontal_scroll_value = (scroll_area_width - pixmap_width) // 2
            # 计算垂直滚动位置
            vertical_scroll_value = (scroll_area_height - pixmap_height) // 2

            # 设置滚动位置
            scroll_area.horizontalScrollBar().setValue(horizontal_scroll_value)
            scroll_area.verticalScrollBar().setValue(vertical_scroll_value)

app = QApplication(sys.argv)
window = MainWindow()
window.show()
app.exec()
