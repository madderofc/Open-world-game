#import pygame
from pygame.locals import *
from OpenGL.GL import *
from OpenGL.GLU import *
import numpy as np
import math
import random

# Game constants
WINDOW_WIDTH = 1200
WINDOW_HEIGHT = 800
WANTED_LEVEL = 0
MAX_WANTED = 5

class Vector3:
    def __init__(self, x=0, y=0, z=0):
        self.x = x
        self.y = y
        self.z = z
    
    def __add__(self, other):
        return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    
    def __sub__(self, other):
        return Vector3(self.x - other.x, self.y - other.y, self.z - other.z)
    
    def distance_to(self, other):
        return math.sqrt((self.x - other.x)**2 + (self.y - other.y)**2 + (self.z - other.z)**2)

class Player:
    def __init__(self):
        self.pos = Vector3(0, 1, 0)
        self.vel = Vector3(0, 0, 0)
        self.health = 100
        self.ammo = 30
        self.angle = 0
        self.speed = 0.5
        self.wanted_level = 0
        self.gun = "AK47"
        self.shooting = False
    
    def update(self, keys):
        # Movement
        if keys[K_w]:
            self.vel.x += math.sin(self.angle) * self.speed
            self.vel.z += math.cos(self.angle) * self.speed
        if keys[K_s]:
            self.vel.x -= math.sin(self.angle) * self.speed
            self.vel.z -= math.cos(self.angle) * self.speed
        if keys[K_a]:
            self.angle += 0.05
        if keys[K_d]:
            self.angle -= 0.05
        
        # Jumping
        if keys[K_SPACE] and self.pos.y <= 1.1:
            self.vel.y = 0.8
        
        # Shooting
        if keys[K_LCTRL] and self.ammo > 0:
            self.shooting = True
            self.ammo -= 1
        else:
            self.shooting = False
        
        # Physics
        self.vel.y -= 0.01  # Gravity
        self.pos = self.pos + self.vel
        self.vel.x *= 0.8  # Friction
        self.vel.z *= 0.8
        
        # Ground collision
        if self.pos.y < 1:
            self.pos.y = 1
            self.vel.y = 0
        
        # World boundaries
        if abs(self.pos.x) > 50:
            self.pos.x = 50 if self.pos.x > 0 else -50
        if abs(self.pos.z) > 50:
            self.pos.z = 50 if self.pos.z > 0 else -50

class NPC:
    def __init__(self, x, z, npc_type="civilian"):
        self.pos = Vector3(x, 1, z)
        self.vel = Vector3(0, 0, 0)
        self.health = 50
        self.npc_type = npc_type  # "civilian" or "police"
        self.angle = random.uniform(0, 2 * math.pi)
        self.move_timer = 0
        self.target = None
        self.is_dead = False
        self.size = 0.5
    
    def update(self, player):
        if self.is_dead:
            return
        
        self.move_timer += 1
        
        if self.npc_type == "police":
            # Police chases player
            dist = self.pos.distance_to(player.pos)
            if dist < 30:
                # Chase player
                dx = player.pos.x - self.pos.x
                dz = player.pos.z - self.pos.z
                dist_xz = math.sqrt(dx**2 + dz**2)
                if dist_xz > 0:
                    self.vel.x = (dx / dist_xz) * 0.3
                    self.vel.z = (dz / dist_xz) * 0.3
                    self.angle = math.atan2(dx, dz)
            else:
                self.vel.x *= 0.9
                self.vel.z *= 0.9
        else:
            # Civilian random walk
            if self.move_timer % 60 == 0:
                self.angle = random.uniform(0, 2 * math.pi)
            
            self.vel.x = math.sin(self.angle) * 0.1
            self.vel.z = math.cos(self.angle) * 0.1
        
        self.pos = self.pos + self.vel
        
        # Boundaries
        if abs(self.pos.x) > 50:
            self.pos.x = 50 if self.pos.x > 0 else -50
        if abs(self.pos.z) > 50:
            self.pos.z = 50 if self.pos.z > 0 else -50
    
    def take_damage(self, damage):
        self.health -= damage
        if self.health <= 0:
            self.is_dead = True
            return True
        return False

class Bullet:
    def __init__(self, pos, angle):
        self.pos = Vector3(pos.x, pos.y, pos.z)
        self.angle = angle
        self.speed = 1.0
        self.lifetime = 100
        self.vel = Vector3(math.sin(angle) * self.speed, 0, math.cos(angle) * self.speed)
    
    def update(self):
        self.pos = self.pos + self.vel
        self.lifetime -= 1
        return self.lifetime > 0
    
    def check_collision(self, npc):
        dist = self.pos.distance_to(npc.pos)
        return dist < 1

class Game:
    def __init__(self):
        pygame.init()
        display = (WINDOW_WIDTH, WINDOW_HEIGHT)
        pygame.display.set_mode(display, DOUBLEBUF | OPENGL)
        pygame.display.set_caption("Open World Game - AK47 Combat")
        
        glEnable(GL_DEPTH_TEST)
        glEnable(GL_LIGHTING)
        glEnable(GL_LIGHT0)
        glEnable(GL_COLOR_MATERIAL)
        glColorMaterial(GL_FRONT_AND_BACK, GL_AMBIENT_AND_DIFFUSE)
        
        gluPerspective(45, (display[0] / display[1]), 0.1, 500.0)
        glTranslatef(0.0, -1, -15)
        
        self.player = Player()
        self.npcs = []
        self.bullets = []
        self.clock = pygame.time.Clock()
        self.running = True
        self.spawn_npcs()
        self.kill_count = 0
        self.police_kills = 0
    
    def spawn_npcs(self):
        # Spawn civilians
        for _ in range(15):
            x = random.uniform(-40, 40)
            z = random.uniform(-40, 40)
            self.npcs.append(NPC(x, z, "civilian"))
        
        # Spawn police
        for _ in range(5):
            x = random.uniform(-40, 40)
            z = random.uniform(-40, 40)
            self.npcs.append(NPC(x, z, "police"))
    
    def draw_cube(self, size=0.5, color=(1, 1, 1)):
        glColor3f(*color)
        glBegin(GL_TRIANGLES)
        vertices = [
            [-size, -size, size], [size, -size, size], [size, size, size], [-size, size, size],
            [-size, -size, -size], [size, -size, -size], [size, size, -size], [-size, size, -size]
        ]
        edges = [
            (0, 1), (1, 2), (2, 3), (3, 0),
            (4, 5), (5, 6), (6, 7), (7, 4),
            (0, 4), (1, 5), (2, 6), (3, 7)
        ]
        for edge in edges:
            for vertex in edge:
                glVertex3f(*vertices[vertex])
        glEnd()
    
    def draw_ground(self):
        glColor3f(0.2, 0.7, 0.2)
        glBegin(GL_QUADS)
        glVertex3f(-100, 0, -100)
        glVertex3f(100, 0, -100)
        glVertex3f(100, 0, 100)
        glVertex3f(-100, 0, 100)
        glEnd()
    
    def draw_player(self):
        glPushMatrix()
        glTranslatef(self.player.pos.x, self.player.pos.y, self.player.pos.z)
        self.draw_cube(0.4, (0, 0, 1))  # Blue cube for player
        glPopMatrix()
    
    def draw_npcs(self):
        for npc in self.npcs:
            if not npc.is_dead:
                glPushMatrix()
                glTranslatef(npc.pos.x, npc.pos.y, npc.pos.z)
                color = (1, 0, 0) if npc.npc_type == "police" else (1, 1, 0)
                self.draw_cube(0.3, color)
                glPopMatrix()
    
    def draw_bullets(self):
        for bullet in self.bullets:
            glPushMatrix()
            glTranslatef(bullet.pos.x, bullet.pos.y, bullet.pos.z)
            self.draw_cube(0.05, (1, 1, 0))
            glPopMatrix()
    
    def draw_ui(self):
        # Switch to 2D mode for text
        glMatrixMode(GL_PROJECTION)
        glPushMatrix()
        glLoadIdentity()
        glOrtho(0, WINDOW_WIDTH, WINDOW_HEIGHT, 0, -1, 1)
        glMatrixMode(GL_MODELVIEW)
        glPushMatrix()
        glLoadIdentity()
        
        # Simple text rendering (basic approach)
        glColor3f(1, 1, 1)
        
        glPopMatrix()
        glMatrixMode(GL_PROJECTION)
        glPopMatrix()
        glMatrixMode(GL_MODELVIEW)
    
    def update(self):
        keys = pygame.key.get_pressed()
        self.player.update(keys)
        
        # Update NPCs
        for npc in self.npcs:
            npc.update(self.player)
        
        # Shooting
        if self.player.shooting:
            bullet = Bullet(self.player.pos, self.player.angle)
            self.bullets.append(bullet)
        
        # Update bullets and check collisions
        dead_bullets = []
        for i, bullet in enumerate(self.bullets):
            if not bullet.update():
                dead_bullets.append(i)
            else:
                for npc in self.npcs:
                    if not npc.is_dead and bullet.check_collision(npc):
                        if npc.take_damage(10):
                            self.kill_count += 1
                            if npc.npc_type == "police":
                                self.police_kills += 1
                            if npc.npc_type == "civilian":
                                self.player.wanted_level = min(5, self.player.wanted_level + 1)
                        dead_bullets.append(i)
                        break
        
        for i in sorted(dead_bullets, reverse=True):
            self.bullets.pop(i)
        
        # Police damage to player
        for npc in self.npcs:
            if npc.npc_type == "police" and not npc.is_dead:
                dist = npc.pos.distance_to(self.player.pos)
                if dist < 2:
                    self.player.health -= 0.5
                    if self.player.health <= 0:
                        self.running = False
    
    def render(self):
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
        glLoadIdentity()
        glTranslatef(0, -2, -15)
        
        self.draw_ground()
        self.draw_player()
        self.draw_npcs()
        self.draw_bullets()
        
        pygame.display.flip()
    
    def run(self):
        while self.running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    self.running = False
                if event.type == KEYDOWN:
                    if event.key == K_ESCAPE:
                        self.running = False
            
            self.update()
            self.render()
            self.clock.tick(60)
        
        pygame.quit()

if __name__ == "__main__":
    game = Game()
    game.run() Open-world-game
