from machine import Pin
from microdot_asyncio import Microdot, send_file
from microdot_asyncio_websocket import with_websocket
from microdot_cors import CORS
import uasyncio as asyncio


from car import Carfactory
from wifi import Wifi

car = Carfactory() # 创建一个汽车

# 定义一个函数 异步函数
async def task1():
    print("task1")
    #i = 0
    while True:
        #i += 1
        #if i % 100 == 0:
        #    print("进来了")
        car.run()
        await asyncio.sleep_ms(50)

# 核心任务
async def task2():
    print("task2")
    # 实例化一个web框架对象
    app = Microdot()
    # 跨域配置
    CORS(app, allowed_origins='*', allow_credentials=True,)

    # 首页
    @app.route('/')
    async def index(request):R 
        return send_file('index.html')

    # 前进
    @app.get('/forward/<int:speed>')
    async def forward(req, speed):
        speedstr = str(speed)
        car.process_command("FORWARD " + speedstr)
        return speedstr + ' speed forward'

    # 停止
    @app.get('/stop')
    async def stop(req):
        car.process_command("STOP")
        return 'stop'

    # 后退
    @app.get('/reverse/<int:speed>')
    async def reverse(req, speed):
        speedstr = str(speed)
        car.process_command("BACKWARD " + speedstr)
        return speedstr + ' speed reverse'

    # 转向
    @app.get('/turnangle/<int:angle>')
    async def turnangle(req, angle):
        anglestr = str(angle)
        car.process_command("TURN " + anglestr)
        return anglestr + ' turnangle'

    # 查询速度
    @app.route('/readopticalcoupler_with_socket')
    @with_websocket
    async def echo(request, ws):
        while True:
            await asyncio.sleep_ms(100)
            # message = await ws.receive()
            message = car.process_command("GETSPEED")
            await ws.send(str(message))

    await app.start_server(host='0.0.0.0', port=80, debug=True, ssl=None)


async def run():
    
    Wifi("tr","12345678").start_ap()
    
    led = Pin(32, Pin.OUT)
    led.value(0)
    # 执行任务1 和任务2
    t1 = asyncio.create_task(task1())
    t2 = asyncio.create_task(task2())
    await asyncio.gather(t1, t2)




    from machine import Pin, PWM, Timer

# 车
class Car:
    def __init__(self, servo, motor, opticalCoupler):
        self.servo = servo                   # 初始化舵机
        self.motor = motor                   # 初始化电机
        self.opticalCoupler = opticalCoupler # 光电耦合器

    def getSpeed(self):
        return self.opticalCoupler.getSpeed()

    def run(self):
        self.opticalCoupler.update()
        
    # 命令处理
    def process_command(self, command):
        if command.find('FORWARD') == 0:
            orderInfo = command.split(" ")
            self.motor.forward(int(orderInfo[1]))
        elif command == 'STOP':
            self.motor.stop()
        elif command.find('BACKWARD') == 0:
            orderInfo = command.split(" ")
            self.motor.backward(int(orderInfo[1]))
        elif command.find('TURN') == 0:
            orderInfo = command.split(" ")
            self.servo.turn(int(orderInfo[1]))
        elif command == 'GETSPEED':
            return self.getSpeed()
        else:
            print("Unknown command")

# 舵机
class Servo:
    def __init__(self, pin, freq):
        self.pin = PWM(Pin(pin, Pin.OUT), freq=freq)
        self.turn(90)

    # 控制转向 0 度 45 度 90 度 135 度 180 度
    def turn(self, angle):
        value = int(500 + (angle / 180) * (2500 - 500))
        self.pin.duty_ns(value * 1000)

# 电机
class Motor:
    def __init__(self, forward_pin, backward_pin, freq):
        # 配置 PWM 引脚，并设置频率
        self.forward_pin = PWM(Pin(forward_pin, Pin.OUT), freq = freq)  
        # 配置 PWM 引脚，并设置频率
        self.backward_pin = PWM(Pin(backward_pin, Pin.OUT), freq = freq)
        self.forward(0)

    # 速度 [0 - 1023]，但太少带不动
    def forward(self, speed):
        self.forward_pin.duty(speed)
        self.backward_pin.duty(0)
    
    def backward(self, speed):
        self.forward_pin.duty(0)
        self.backward_pin.duty(speed)

    def stop(self):
        self.forward_pin.duty(0)
        self.backward_pin.duty(0)


# 光电耦合元件
class OpticalCoupler:
    count = 0 # 光耦传感器经过光栅累计（下面代码每秒清零）
    speed = 0 # 光耦传感器测速
    timer = 0
    time = 0

    def __init__(self, optical_coupler_pin):
        # 创建一个Pin对象，使用指定的引脚号，并将其设置为输入模式
        self.optical_coupler = Pin(optical_coupler_pin, Pin.IN)
        self.optical_coupler.irq(trigger=Pin.IRQ_FALLING,handler=self.interruption_handler)

        self.timer = Timer(0)
        self.timer.init(period=1000, mode=Timer.PERIODIC, callback=self.handleInterrupt)

    # 循环更新速度
    def update(self):
        if self.time > 0:
            self.speed = self.count / 18.0 / 21 * 6.2 * 3.14
            self.time = 0
            self.count = 0

    # 读取速度
    def getSpeed(self):
        return self.speed


    # 光耦中断程序
    def interruption_handler(self, pin):
        # print("光耦中断", self.count)
        self.count += 1

    # 时钟中断
    def handleInterrupt(self, timer):
        # print("时钟中断", self.time)
        self.time += 1



# 汽车工厂
def Carfactory():
    servo = Servo(18, 50)
    motor = Motor(33, 25, 1000)
    opticalCoupler = OpticalCoupler(23)
    car = Car(servo, motor, opticalCoupler)
    return car

