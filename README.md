import pygame
import math
import sys
import os
import assist
import time
import tools
from RealtimeSTT import AudioToTextRecorder


class AppCircle:
    def __init__(self, center, app_index, screen_size):
        self.center = center
        self.radius = min(screen_size) // 10
        self.app_index = app_index
        self.image = self.load_image(app_index)
        self.text = f'App {app_index}'

    def load_image(self, app_index):
        path = f'./apps/app_{app_index}/app_{app_index}.png'
        try:
            img = pygame.image.load(path)
            img = pygame.transform.scale(img, (2 * self.radius, 2 * self.radius))
            return img
        except FileNotFoundError:
            print(f"Image file not found: {path}")
            return None

    def draw(self, screen):
        if self.image:
            top_left = (self.center[0] - self.radius, self.center[1] - self.radius)
            screen.blit(self.image, top_left)
        else:
            pygame.draw.circle(screen, (255, 255, 255), self.center, self.radius)
            font = pygame.font.Font(None, 32)
            text_surface = font.render(self.text, True, (0, 0, 0))
            text_rect = text_surface.get_rect(center=self.center)
            screen.blit(text_surface, text_rect)

    def is_clicked(self, pos):
        return math.hypot(pos[0] - self.center[0], pos[1] - self.center[1]) <= self.radius


def create_circles(screen_size):
    circles = []
    num_circles = 8
    angle_step = 360 / num_circles
    center_x, center_y = screen_size[0] // 2, screen_size[1] // 2
    radius = min(screen_size) // 2.6

    for i in range(num_circles):
        angle = math.radians(angle_step * i)
        x = int(center_x + radius * math.cos(angle))
        y = int(center_y + radius * math.sin(angle))
        circles.append(AppCircle((x, y), i + 1, screen_size))
    return circles


def apply_blur_ring_and_text(screen, text, blue_ring_thickness=100):
    blur_surface = pygame.Surface(screen.get_size(), pygame.SRCALPHA)
    blur_surface.fill((0, 0, 0, 128))  # Semi-transparent black for blur effect
    screen.blit(blur_surface, (0, 0))

    ring_surface = pygame.Surface(screen.get_size(), pygame.SRCALPHA)
    center = (screen.get_width() // 2, screen.get_height() // 2)
    outer_radius = screen.get_width() // 2
    inner_radius = outer_radius - blue_ring_thickness

    for i in range(outer_radius, inner_radius, -5):  # Reduce steps for better performance
        alpha = int(128 * (i - inner_radius) / blue_ring_thickness)
        pygame.draw.circle(ring_surface, (173, 216, 230, alpha), center, i)

    screen.blit(ring_surface, (0, 0))

    font = pygame.font.Font(None, 36)
    margin = screen.get_width() // 4
    words = text.split()
    lines = []
    current_line = []
    for word in words:
        test_line = current_line + [word]
        test_surface = font.render(' '.join(test_line), True, (255, 255, 255))
        if test_surface.get_width() <= screen.get_width() - 2 * margin:
            current_line = test_line
        else:
            lines.append(' '.join(current_line))
            current_line = [word]
    lines.append(' '.join(current_line))

    total_height = sum(font.render(line, True, (255, 255, 255)).get_height() for line in lines)
    start_y = (screen.get_height() - total_height) // 2
    for i, line in enumerate(lines):
        text_surface = font.render(line, True, (255, 255, 255))
        text_rect = text_surface.get_rect(center=(screen.get_width() // 2, start_y + i * 40))
        screen.blit(text_surface, text_rect)

    pygame.display.update()


def run_voice_assistant(recorder, screen, background, circles, last_check):
    current_time = pygame.time.get_ticks()
    if current_time - last_check < 100:  # Check every 100ms
        return last_check

    current_text = recorder.text()
    if current_text and ("jarvis" in current_text.lower() or "alexa" in current_text.lower()):
        apply_blur_ring_and_text(screen, current_text)
        recorder.stop()

        response = assist.ask_question_memory(current_text)
        speech = response.split("#")[0]
        apply_blur_ring_and_text(screen, response)

        assist.TTS(speech)
        if len(response.split("#")) > 1:
            tools.parse_command(response.split("#")[1])
        recorder.start()

    return current_time


def run_home_screen(screen):
    screen_size = screen.get_size()
    background = pygame.image.load('./resources/background.jpg')
    background = pygame.transform.scale(background, screen_size)

    circles = create_circles(screen_size)
    recorder = AudioToTextRecorder(
        spinner=False,
        model="tiny.en",
        language="en",
        post_speech_silence_duration=0.1,
        silero_sensitivity=0.4,
    )
    recorder.start()

    running = True
    last_check = 0

    while running:
        updated = False
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                recorder.stop()
                pygame.quit()
                sys.exit()
            elif event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                for circle in circles:
                    if circle.is_clicked(event.pos):
                        app_index = circles.index(circle) + 1
                        app_module_name = f'apps.app_{app_index}.app_{app_index}'
                        try:
                            mod = __import__(app_module_name, fromlist=[''])
                            mod.run(screen)
                            updated = True
                        except ImportError:
                            print(f"Failed to load app module: {app_module_name}")

        screen.blit(background, (0, 0))
        for circle in circles:
            circle.draw(screen)
        last_check = run_voice_assistant(recorder, screen, background, circles, last_check)

        pygame.display.flip()
        pygame.time.delay(1)


if __name__ == '__main__':
    pygame.init()
    screen = pygame.display.set_mode((1080, 1080))
    run_home_screen(screen)

