"""
可视化界面
编辑时间：2022.10.26.10.12
"""
import subprocess

print('请耐心等待几秒，系统正在启动。。')

import threading
import tkinter
import tkinter.messagebox
import ctypes
import inspect
import time
import cv2
import dlib
import os
from os import mkdir
from os.path import isdir
from threading import Thread
from _datetime import datetime

"""一些前置操作"""
# 创建窗口
win = tkinter.Tk()
win.title('运动捕获平台')
# win.geometry('640x480')
# 信息框
txtInfoST = tkinter.Text(win, width=55, height=14)
scInfoSC = tkinter.Scrollbar()
scInfoSC.config(command=txtInfoST.yview)
txtInfoST.config(yscrollcommand=scInfoSC.set)

"""初始化视窗的一切文本，包括设置默认值（value=xx）"""
# 切勿移动该部分代码，避免后续的函数无法获取到默认值
strCam = tkinter.IntVar(value=0)  # 选择摄像头，通常0是一台设备的默认摄像头
strSpeed = tkinter.IntVar(value=1)  # 录制倍速，默认一倍速录制
strCatchTime = tkinter.IntVar(value=5)  # 录制时长，默认为5秒
strTimeToSleep = tkinter.IntVar(value=3)  # 捕获间隔，默认间隔3秒
strSSTV = tkinter.IntVar(value=5)  # 运动敏感度，默认设置为5是一个测试出来的比较刚好的敏感度。此项数值越小，越细微的运动会被捕获
catchMark = tkinter.IntVar()  # 捕获状态，默认为0。因为其用在radiation，它会默认选择第一项，所以可以直接不用设置默认值

"""函数功能区（所有的函数被整合在此处）"""


def asyncRaise(tID, tType):
    """进程状态设置，tID是进程id，tType是进程类型"""
    tID = ctypes.c_long(tID)  # 获取进程id长度
    if not inspect.isclass(tType):
        tType = type(tType)  # 如果进程状态不是type数据类型，转变为type
    # 设置对应进程为指定状态【ctypes.py_object(tType)】，返回值为1则设置成功
    res = ctypes.pythonapi.PyThreadState_SetAsyncExc(tID, ctypes.py_object(tType))
    if res == 0:
        # 当进程id非法时
        raise ValueError("进程id为非法")
    elif res != 1:
        # 当进程状态设置失败时
        ctypes.pythonapi.PyThreadState_SetAsyncExc(tID, None)
        raise SystemError("进程设置异步失败")


def stopThread(thread):
    """终止进程"""
    asyncRaise(thread.ident, SystemExit)


def recordVid(cap, fps, cTime, width, height):
    """录制视频"""

    def write():
        """视频文件写入"""
        while cap.isOpened():
            ret, frame = cap.read()
            if ret:  # 写入视频文件
                vidFile.write(frame)
        vidFile.release()

    now = str(datetime.now())[:19].replace(':', '.')  # 当前日期时间
    dirName = now[:10]  # 目录名
    vidFileName = dirName + '/' + now[11:] + '.mp4'  # 视频文件名
    if not isdir(dirName):  # 创建目录
        mkdir(dirName)
    # 创建视频文件
    #                                                                解码器（最好是mp4v），帧速， 视频宽度、 高度
    vidFile = cv2.VideoWriter(vidFileName, cv2.VideoWriter_fourcc('m', 'p', '4', 'v'), fps, (width, height))
    t = Thread(target=write)
    t.start()
    time.sleep(cTime)  # 利用time.sleep设置录制时间
    stopThread(t)  # 利用进程终止来结束捕获，这样的话不用关闭摄像头


def faceRecord(cap):
    """识别人脸"""
    detector = dlib.get_frontal_face_detector()  # 导入人脸识别包
    caught, frameLWPCV = cap.read()  # 读取视频流
    if caught:
        grayLWPCV = cv2.cvtColor(frameLWPCV, cv2.COLOR_BGR2GRAY)  # 转为灰度图
        rects = detector(grayLWPCV)  # 识别到的人脸将会用矩形框出
        if len(rects) != 0:
            txtInfoST.insert('1.0', '监测到人脸-> ' + datetime.now().strftime('%H:%M:%S') + '\n')
            f, frame = cap.read()  # 将摄像头中的一帧图片数据保存
            now = str(datetime.now())[:19].replace(':', '.')  # 当前日期时间
            dirName = now[:10]  # 目录名
            if not isdir(dirName):  # 创建目录
                mkdir(dirName)
            cv2.imwrite(dirName + '/Face.' + now[11:] + '.jpg', frame)  # 将图片保存为本地文件


def activeProcess(camera, catchID, fps, cTime, timeToSleep, sensitiveness):
    """捕获摄像头处理"""
    """设置监测范围
    零点坐标在左上角，设置为（0，0，0，0）可以框住全部内容，
              设置为（100，450，100，300）可以在中间留一个大面积的矩形
    """
    rectX = 0  # 矩形最左点x坐标
    rectXCols = 0  # 矩形x轴上的长度
    rectY = 0  # 矩形最上点y坐标
    rectYCols = 0  # 矩形y轴上的长度
    KeyFrame = 24  # 取关键帧的间隔数。若设置摄像头帧率为24FPS、关键帧间隔为12，则每半秒取一个关键帧
    counter = 1  # 取帧计数器
    preFrame = None  # 总是取视频流前一帧做为背景相对下一帧进行比较

    # 判断摄像头是否打开
    if not camera.isOpened():
        txtInfoST.insert('1.0', '摄像头打开失败！\n')
    else:

        width = int(camera.get(cv2.CAP_PROP_FRAME_WIDTH))
        height = int(camera.get(cv2.CAP_PROP_FRAME_HEIGHT))
        txtInfoST.insert('1.0', '摄像头画面尺寸（高x宽）:' + str(height) + 'x' + str(width) + '\n')

        if rectXCols == 0:
            rectXCols = width - rectX
        if rectYCols == 0:
            rectYCols = height - rectY
        start_time = datetime.now().strftime('%H:%M:%S')
        txtInfoST.insert('1.0', '开始处理捕获画面-> ' + start_time + '，摄像头已启动\n')

        while True:
            caught, frameLWPCV = camera.read()  # 读取视频流
            if caught:
                if counter % KeyFrame == 0:
                    grayLWPCV = cv2.cvtColor(frameLWPCV, cv2.COLOR_BGR2GRAY)  # 转为灰度图
                    # 用矩形框框住监测区域，此处设置的数值刚好足够在测试机上框住一个小一圈的矩形框
                    grayLWPCV = grayLWPCV[rectY: rectY + rectYCols, rectX: rectX + rectXCols]
                    cv2.rectangle(frameLWPCV, (rectX, rectY),
                                  (rectX + rectXCols, rectY + rectYCols),
                                  (0, 255, 0), 2)
                    # 显示视频播放窗口，开启消耗时间大概是3倍
                    # cv2.imshow('lwpCVWindow', frameLWPCV)
                    grayLWPCV = cv2.GaussianBlur(grayLWPCV, (21, 21), 0)  # 对灰度图高斯模糊处理，减少硬件占用
                    if preFrame is None:
                        # 关键帧为预设的None时，将第一个灰度图设为关键帧
                        preFrame = grayLWPCV
                    else:
                        # 上一关键帧（preFrame）和当前关键帧（grayLWPCV）的差异图；
                        imgDelta = cv2.absdiff(preFrame, grayLWPCV)
                        # 将差异图普通阈值化处理
                        # 此处将其设置为THRESH_BINARY，使灰度图变为仅有黑色和白色的图片
                        # 该处理将极大提高绘制轮廓图后并计算轮廓面积时的效率
                        threshBinary = cv2.threshold(imgDelta, 25, 255, cv2.THRESH_BINARY)[1]
                        # 将差异图膨胀处理，减少硬件资源占用（腐蚀次数经测试选择2次最为合适）
                        threshBinary = cv2.dilate(threshBinary, None, iterations=2)
                        # 绘制轮廓图（后续不需要每条轮廓对应的属性，故只需要轮廓图）
                        # contours存储了轮廓图各个点的坐标轴
                        contours = cv2.findContours(threshBinary, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)[0]
                        for x in contours:
                            if cv2.contourArea(x) < sensitiveness:
                                # 若轮廓图的面积小于预设置的面积（即面向用户声称的“运动敏感度”）
                                # 则判定摄像头捕获的画面未发生运动
                                continue
                            else:

                                if catchID == 0:
                                    txtInfoST.insert('1.0', '监测到运动-> ' + datetime.now().strftime('%H:%M:%S') + '\n')
                                    # 创建进程以便同时进行运动和人脸检测
                                    threads = [Thread(target=faceRecord(camera)),
                                               Thread(target=recordVid(camera, fps, cTime, width, height))]
                                    # print(threads)
                                    for y in threads:
                                        y.start()
                                    txtInfoST.insert('1.0', '结束捕获\n')
                                elif catchID == 1:
                                    txtInfoST.insert('1.0', '监测到画面发生运动-> ' + datetime.now().strftime('%H:%M:%S') + '\n')
                                    recordVid(camera, fps, cTime, width, height)
                                    txtInfoST.insert('1.0', '录制结束\n')
                                elif catchID == 2:
                                    faceRecord(camera)

                                time.sleep(timeToSleep)  # 暂停任意时间后再进行运动检测
                                break

                        preFrame = grayLWPCV
                counter += 1
        # cv2.destroyAllWindows()  # 与上面的imshow对应


#         传入(函数名，*参数)
def threadIt(func, *args):
    """创建线程"""
    t = threading.Thread(target=func, args=args)
    # 守护线程
    t.setDaemon(True)
    # 启动
    t.start()


def catchActive():
    """按下”启用捕获”按钮后"""

    """弹出“停止运行”按钮"""

    def stopCatch():
        """当“停止运行”被按下后"""
        camera.release()
        txtInfoST.insert('1.0', '\n\n此次监控已结束\n->>请自行确认是否为主动关闭\n\n')
        # tkinter.messagebox.* 所弹出的窗口无法在Linux系统下被关闭
        # tkinter.messagebox.showwarning(title='监控已停止运行！', message='请自行确认是否为主动关闭')

    btnStop = tkinter.Button(win, width=6, text='停止运行', command=lambda: threadIt(stopCatch))
    btnStop.grid(row=6, column=4, columnspan=2)

    # 获取状态
    SSTV = strSSTV.get()
    SSTV *= 1000
    cam = strCam.get()
    speed = strSpeed.get()
    speed *= 24
    catchTime = strCatchTime.get()
    timeToSleep = strTimeToSleep.get()
    mark = catchMark.get()
    txtInfoST.insert('1.0', '程序已启动，请等待数秒..\n')
    # 启动程序
    camera = cv2.VideoCapture(cam)
    activeProcess(camera, mark, speed, catchTime, timeToSleep, SSTV)


# def fileRead():
#     """打开视频文件"""
#     vidDirectory = r''
#     os.startfile(vidDirectory)


def fileRead():
    """打开视频文件"""
    vidDirectory = r''
    try:
        os.startfile(vidDirectory)
    except:
        subprocess.Popen(['xdg-open', vidDirectory])


def readMe():
    """用户使用说明"""
    txtInfoST.insert('1.0', '\n\n'
                            '用户使用说明：\n'
                            '1、敏感度：运动检测敏感度，\n'
                            '\t其数字越小越能捕获细微运动；\n'
                            '2、选择摄像头：通常0是一台设备的默认摄像头，\n'
                            '\t可根据实际硬件情况自由变更；\n'
                            '3、录制倍速：默认1倍速，\n'
                            '\t可根据需求自由调节；\n'
                            '4、捕获时长：每次检测到运动时录制的视频片段长度；\n'
                            '5、捕获间隔：本轮检测结束后，间隔多久进行下一次检测。\n'
                            '\n\n')


"""界面元素"""
# 按钮
btnActive = tkinter.Button(win, width=10, text='启用捕获', command=lambda: threadIt(catchActive))
btnFile = tkinter.Button(win, width=15, text='打开捕获目录', command=lambda: threadIt(fileRead))
# 手动输入选择摄像头
txtCam = tkinter.Label(win, width=10, text='选择摄像头：')
enCam = tkinter.Entry(win, width=8, textvariable=strCam)
# 调整录制画面的倍数
txtSpeed = tkinter.Label(win, width=10, text='录制倍速：')
enSpeed = tkinter.Entry(win, width=8, textvariable=strSpeed)
# 设置录制时长
txtCatchTime = tkinter.Label(win, width=12, text='捕获时长(秒)：')
enCatchTime = tkinter.Entry(win, width=8, textvariable=strCatchTime)
# 相隔多少秒检测一次
txtTimeToSleep = tkinter.Label(win, width=12, text='捕获间隔(秒)：')
enTimeToSleep = tkinter.Entry(win, width=8, textvariable=strTimeToSleep)
# 运动敏感度（数字越小，越细微的运动将会被捕获）
txtSSTV = tkinter.Label(win, width=28, text='敏感度(数字越小越能捕获细微运动)：')
enSSTV = tkinter.Entry(win, width=8, textvariable=strSSTV)
# 选择录制模式
rbBoth = tkinter.Radiobutton(win, text='同时检测', variable=catchMark, value=0)
rbActive = tkinter.Radiobutton(win, text='仅运动', variable=catchMark, value=1)
rbFace = tkinter.Radiobutton(win, text='仅人脸', variable=catchMark, value=2)
# read me
btnReadMe = tkinter.Button(win, width=15, text='用户使用说明', command=lambda: threadIt(readMe))

"""布局（row=行，column=列，columnspan=占用了多少列）"""
# 第一行
btnActive.grid(row=1, column=1)
txtSSTV.grid(row=1, column=2, columnspan=3)
enSSTV.grid(row=1, column=5)
# 第二行
txtCam.grid(row=2, column=1)
enCam.grid(row=2, column=2)
txtSpeed.grid(row=2, column=3)
enSpeed.grid(row=2, column=4)
# 第三行
txtCatchTime.grid(row=3, column=1)
enCatchTime.grid(row=3, column=2)
txtTimeToSleep.grid(row=3, column=3)
enTimeToSleep.grid(row=3, column=4)
# 第四行
rbBoth.grid(row=4, column=1)
rbActive.grid(row=4, column=2)
rbFace.grid(row=4, column=3)
# 第五行
txtInfoST.grid(row=0, column=1, columnspan=6)
# 第六行
btnFile.grid(row=6, column=1, columnspan=2)
# 第七行
btnReadMe.grid(row=7, column=4, columnspan=2)

# 文本插入测试
# txtInfoST.insert('1.0', '摄像头启动\n')
# txtInfoST.insert('1.0', '程序执行=====》\n')


win.mainloop()  # 展示窗口
