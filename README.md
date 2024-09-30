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
