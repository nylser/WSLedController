#!/usr/bin/python3
# -*- coding: utf-8 -*-

import json
import pigpio
import websockets
import asyncio
import time
import calendar
import threading
import copy

colors = [None] * 3
pi = None

rgb_pins = [17, 22, 24]

CONST_MODE = 0
FADE_MODE = 1

lastupdate = time.time() * 1000
min_update_time = 1

mode = CONST_MODE

lock = asyncio.Lock()
counter_lock = asyncio.Lock()
connected = dict()

fade_thread = None

def fade_step(from_color, to_color):
    for i, c in enumerate(from_color):
        if to_color[i] < c:
            from_color[i] = c-1
        elif to_color[i] > c:
            from_color[i] = c+1
    set_colors(from_color)

class FadeThread(threading.Thread):
    def __init__(self, color1, color2):
        super(FadeThread, self).__init__()
        self._stop = threading.Event()
        self.color1 = color1
        self.color2 = color2
        self.colorupdate = 0
        self.new_color = 0
        self.current_color = copy.copy(colors)

    def run(self):
        while not self.stopped():
            self.fade_color(self.color1)
            self.fade_color(self.color2)
            if self.colorupdate > 0:
                if self.colorupdate == 1:
                    self.color1 = self.new_color
                    self.colorupdate = 0
                    self.fade_color(self.color1)
                elif self.colorupdate == 2:
                    self.color2 = self.new_color
                    self.colorupdate = 0
                    self.fade_color(self.color2)


    def update_color1(self, color):
        self.colorupdate = 1
        self.new_color = color

    def update_color2(self, color):
        self.colorupdate = 2
        self.new_color = color

    def fade_color(self, to_color):
        while (self.current_color[0] != to_color[0] or self.current_color[1] != to_color[1] or self.current_color[2] != to_color[2]):
            if self.colorupdate > 0 or self.stopped():
                break
            fade_step(self.current_color, to_color)
#            time.sleep(0.05)

    def stop(self):
        self._stop.set()

    def stopped(self):
        return self._stop.is_set()

def update_colors(r=0, g=0, b=0):
    global colors
    colors[0] = r
    colors[1] = g
    colors[2] = b

def set_colors(color=colors):
    for index, pin in enumerate(rgb_pins):
        pi.set_PWM_dutycycle(pin, color[index])

async def led_control(websocket, path):
    global connected, mode, fade_thread
    if path == "/get":
        uuid = await websocket.recv()
        connected[uuid] = websocket
        try:
            with(await lock):
                global mode, colors
                await websocket.send(json.dumps({"mode": mode, "color": colors}))

            while websocket.open:
                await asyncio.sleep(1)
                await websocket.send(json.dumps({"mode": mode, "color": colors}))
            print("socket closed")
        finally:
             del connected[uuid]

    elif path == "/set":
        uuid = await websocket.recv()
        while True:
            rgb = await websocket.recv()
            if mode == CONST_MODE:
                r, g, b = rgb.split(",", 3)
                update_colors(int(r), int(g), int(b))
                set_colors()
            elif mode == FADE_MODE:
                r, g, b = rgb.split(",", 6)
                fade_thread.update_color2((int(r), int(g), int(b)))
                update_colors(int(r), int(g), int(b))
            global lastupdate
            try:
                if len(connected) > 1 and time.time() * 1000 - lastupdate > min_update_time:
                    await asyncio.wait([ws.send(json.dumps({"mode": mode, "color": colors})) for uuid_spec, ws in connected.items() if uuid_spec != uuid and ws.open])
                    lastupdate = time.time() * 1000

            except Exception as err:
                pass

    elif path == "/modeset":
        while True:
            new_mode = await websocket.recv()
            if new_mode == "const" and mode != CONST_MODE:
                fade_thread.stop()
                try:
                    fade_thread.join()
                except:
                    pass
                set_colors()
                mode = CONST_MODE
                pass
            elif new_mode == "fade" and mode != FADE_MODE:
                fade_thread = FadeThread((0,0,0), (copy.copy(colors)))
                fade_thread.start()
                mode = FADE_MODE

def init_stuff():
    global pi
    pi = pigpio.pi()
    update_colors(0, 0, 0)
    set_colors()

if __name__ == "__main__":
    init_stuff()
    print("PiGPIO LEDs initialized\n Starting eventloop")
    start_server = websockets.serve(led_control, '', 8765)

    asyncio.get_event_loop().run_until_complete(start_server)
    asyncio.get_event_loop().run_forever()
