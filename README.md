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
    """Apply a simplified blur effect to the screen, draw a subtle transparent blue ring, and overlay text."""
    # Create a semi-transparent surface for the blur effect
    blur_surface = pygame.Surface(screen.get_size(), pygame.SRCALPHA)
    blur_surface.fill((0, 0, 0, 128))

    # Draw the blurred screen back onto the main screen
    screen.blit(blur_surface, (0, 0))

    # Create a transparent surface for the gradient ring effect
    ring_surface = pygame.Surface(screen.get_size(), pygame.SRCALPHA)
    center = (screen.get_width() // 2, screen.get_height() // 2)
    outer_radius = screen.get_width() // 2
    inner_radius = outer_radius - blue_ring_thickness

    # Draw a gradient ring
    for i in range(outer_radius, inner_radius, -1):
        alpha = int(128 * (i - inner_radius) / blue_ring_thickness)
        pygame.draw.circle(ring_surface, (173, 216, 230, alpha), center, i)

    # Overlay the ring surface onto the screen
    screen.blit(ring_surface, (0, 0))

    # Draw text
    font = pygame.font.Font(None, 36)
    text_surface = font.render(text, True, (255, 255, 255))
    text_rect = text_surface.get_rect(center=(screen.get_width() // 2, screen.get_height() // 2))
    screen.blit(text_surface, text_rect)

    pygame.display.update()


def run_app(app_index, screen):
    try:
        app_module_name = f'apps.app_{app_index}.app_{app_index}'
        print(f"Attempting to launch: {app_module_name}")
        mod = __import__(app_module_name, fromlist=[''])
        mod.run(screen)
    except ImportError as e:
        print(f"Error: Unable to load app {app_index}. Module not found: {e}")
    except AttributeError:
        print(f"Error: App {app_index} does not have a 'run' function.")
    except Exception as e:
        print(f"Unexpected error while running app {app_index}: {e}")


def run_voice_assistant(circles, screen, background):
    recorder = AudioToTextRecorder(spinner=False, model="tiny.en", language="en",
                                   post_speech_silence_duration=0.1, silero_sensitivity=0.4)
    recorder.start()
    hot_words = ["jarvis", "alexa"]
    skip_hot_word_check = False

    while True:
        current_text = recorder.text()
        if any(hot_word in current_text.lower() for hot_word in hot_words) or skip_hot_word_check:
            if current_text:
                print(f"User: {current_text}")
                apply_blur_ring_and_text(screen, current_text, blue_ring_thickness=100)

                recorder.stop()
                response = assist.ask_question_memory(current_text)
                print(response)
                speech = response.split('#')[0]

                if len(response.split('#')) > 1:
                    command = response.split('#')[1]
                    if "open app" in command:
                        try:
                            app_index = int(command.split()[-1])
                            run_app(app_index, screen)
                        except ValueError:
                            print("Error: Invalid app index in command.")
                        except Exception as e:
                            print(f"Unexpected error: {e}")

                done = assist.TTS(speech)
                recorder.start()


def run_home_screen(screen):
    screen_size = screen.get_size()
    background = pygame.image.load('./resources/background.jpg')
    background = pygame.transform.scale(background, screen_size)
    circles = create_circles(screen_size)

    running = True

    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            elif event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                for circle in circles:
                    if circle.is_clicked(event.pos):
                        app_index = circles.index(circle) + 1
                        run_app(app_index, screen)

        # Draw home screen
        screen.blit(background, (0, 0))
        for circle in circles:
            circle.draw(screen)
        pygame.display.flip()
        pygame.time.delay(1)


if __name__ == '__main__':
    pygame.init()
    screen = pygame.display.set_mode((1080, 1080))

    # Start voice assistant in the same loop as the home screen
    try:
        run_home_screen(screen)
        run_voice_assistant([], screen, None)
    except KeyboardInterrupt:
        pygame.quit()
        sys.exit()


