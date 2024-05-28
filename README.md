import sys

import numpy as np
from PySide6.QtCore import QSize, Qt
from PySide6.QtGui import QAction, QIcon, QKeySequence, QImage, QPixmap
from PySide6.QtWidgets import (
    QApplication,
    QCheckBox,
    QLabel,
    QMainWindow,
    QStatusBar,
    QToolBar, QWidget, QVBoxLayout, QFileDialog, QGridLayout, QScrollArea, QSizePolicy,
)
from matplotlib import pyplot as plt
from osgeo import gdal  # 或者直接用import gdal



class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()

        self.setWindowTitle("Our App")
        self.base_image_path = None
        self.drone_image_path = None
        self.setGeometry(100, 100, 1200, 600)  # 调整窗口大小以适应两个图像视图

        self.base_image_label=QLabel()#用于显示基准影像的Qlabel
        self.drone_image_label=QLabel()
        #自动适应图像的大小
        self.base_image_label.setScaledContents(True)
        self.drone_image_label.setScaledContents(True)
       
        #使用QScrollArea类创建一个滚动区域对象，并将其命名为self.base_scroll_area
        self.base_scroll_area = QScrollArea(self)
        self.base_scroll_area.setWidget(self.base_image_label)
        self.drone_scroll_area = QScrollArea(self)
       
        self.drone_scroll_area.setWidget(self.drone_image_label)
#创建一个打开的菜单
        menu = self.menuBar()
        file_menu = menu.addMenu("打开")
        conduct_menu=menu.addMenu("处理")
        #file_menu.addAction(button_action)

        file_menu.addSeparator()
        file_submenu1 = file_menu.addMenu("打开基准影像")
        file_menu.addSeparator()
        file_submenu2=file_menu.addMenu("打开待配准的无人机影像")

        conduct_menu.addSeparator()
        conduct_menu1=conduct_menu.addMenu("预处理")
        conduct_menu.addSeparator()
        conduct_menu1=conduct_menu.addMenu("   ")

#为菜单连接槽函数
        button_action1=QAction("tiff、png、jpg",self)
        button_action1.triggered.connect(self.open_base_image)
        #button_action1.setCheckable(True)
        file_submenu1.addAction(button_action1)

        button_action2 = QAction("tiff、png、JPG", self)
        button_action2.triggered.connect(self.open_drone_image)
        #button_action2.setCheckable(True)
        file_submenu2.addAction(button_action2)

        button_action_yuchuli=QAction("  ",self)
        button_action_yuchuli.triggered.connect(self.yuchuli)



        # 创建主窗口布局
        main_layout = QGridLayout()

        # 将 QLabel 放置在 QScrollArea 中
        base_scroll_area = QScrollArea()
        base_scroll_area.setWidget(self.base_image_label)
        main_layout.addWidget(base_scroll_area, 0, 0)  # 基准影像在左侧

        drone_scroll_area = QScrollArea()
        drone_scroll_area.setWidget(self.drone_image_label)
        main_layout.addWidget(drone_scroll_area, 0, 1)  # 无人机影像在右侧

        self.base_image_label.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Expanding)
        self.drone_image_label.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Expanding)

        central_widget = QWidget()
        central_widget.setLayout(main_layout)
        self.setCentralWidget(central_widget)

#定义打开影像的槽函数
    #打开基准影像
    def open_base_image(self):
        file_path, _ = QFileDialog.getOpenFileName(
            self, "Open Image", "", "Image Files (*.png *.jpg *.tif *.tiff);;All Files (*)"
        )
        if file_path:
            self.base_image_path = file_path

            if file_path.endswith((".tif", ".tiff")):
                # 使用 GDAL 打开图像
                dataset = gdal.Open(self.base_image_path)
                width = dataset.RasterXSize
                height = dataset.RasterYSize
                bands = dataset.RasterCount

                # 读取所有波段数据
                image_data = dataset.ReadAsArray().reshape(height, width, bands)
                # 处理可能存在的单波段图像数据类型问题
                if bands == 1:
                    image_data = image_data.astype(np.uint8)

                    # 将图像数据转换为 QImage
                if bands == 1:
                    image = QImage(image_data.flatten(), width, height, QImage.Format_Grayscale8)
                elif bands == 3:
                    image = QImage(image_data.tobytes(), width, height, QImage.Format_RGB888)
                else:
                    print("不支持的波段数")
                    return

            else:  # 打开 PNG 或 JPG 图像
                image = QImage(self.base_image_path)

            # 将 QImage 转换为 QPixmap 并显示在 QLabel 中
            pixmap = QPixmap.fromImage(image)
            self.base_image_label.setPixmap(pixmap)
            self.base_image_label.adjustSize()
            self.base_scroll_area.widget().resize(self.base_image_label.size())

    def open_drone_image(self):
        file_path, _ = QFileDialog.getOpenFileName(
            self, "Open Image", "", "Image Files (*.png *.jpg *.tif *.tiff);;All Files (*)"
        )
        if file_path:
            self.drone_image_path = file_path

            if file_path.endswith((".tif", ".tiff")):
                # 使用 GDAL 打开图像
                dataset = gdal.Open(self.drone_image_path)
                width = dataset.RasterXSize
                height = dataset.RasterYSize
                bands = dataset.RasterCount

                # 读取所有波段数据
                image_data = dataset.ReadAsArray().reshape(height, width, bands)
                # 处理可能存在的单波段图像数据类型问题
                if bands == 1:
                    image_data = image_data.astype(np.uint8)

                    # 将图像数据转换为 QImage
                if bands == 1:
                    image = QImage(image_data.flatten(), width, height, QImage.Format_Grayscale8)
                elif bands == 3:
                    image = QImage(image_data.tobytes(), width, height, QImage.Format_RGB888)
                else:
                    print("不支持的波段数")
                    return

            else:  # 打开 PNG 或 JPG 图像
                image = QImage(self.drone_image_path)

            # 将 QImage 转换为 QPixmap 并显示在 QLabel 中
            pixmap = QPixmap.fromImage(image)
            self.drone_image_label.setPixmap(pixmap)
            self.drone_image_label.adjustSize()
            self.drone_scroll_area.widget().resize(self.base_image_label.size())


#影响预处理
    def yuchuli(self):
        print("open")



#创建一个文件选择器
class FileSelector(QWidget):
    def __init__(self):
        super().__init__()

        self.setWindowTitle("文件选择器")

        self.file_path = None
        self.label = QLabel("请选择文件")

        layout = QVBoxLayout()
        layout.addWidget(self.label)

        self.setLayout(layout)

    def open_file_dialog(self):
        # 打开文件选择对话框
        file_dialog = QFileDialog(self)
        file_dialog.setFileMode(QFileDialog.ExistingFile)
        file_dialog.setNameFilter("所有文件 (*);;TIFF 文件 (*.tiff)")  # 可以根据需要修改文件过滤器

        if file_dialog.exec():
            # 获取选择的文件路径
            self.file_path = file_dialog.selectedFiles()[0]
            self.label.setText(f"已选择文件: {self.file_path}")
            return self.file_path
        return None  # 如果没有选择文件，返回 None

app = QApplication(sys.argv)

window = MainWindow()
window.show()

app.exec()
