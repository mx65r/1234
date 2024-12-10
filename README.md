import pygame
import math
import sys
import importlib
from RealtimeSTT import AudioToTextRecorder
import assist


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


def load_background(path, screen_size):
    try:
        background = pygame.image.load(path)
        return pygame.transform.scale(background, screen_size)
    except FileNotFoundError:
        print("Background image not found. Using solid color.")
        surface = pygame.Surface(screen_size)
        surface.fill((50, 50, 50))
        return surface


def draw_home_screen(screen, background, circles):
    screen.blit(background, (0, 0))
    for circle in circles:
        circle.draw(screen)


def apply_blur_ring_and_text(screen, text, blue_ring_thickness=100):
    blur_surface = pygame.Surface(screen.get_size(), pygame.SRCALPHA)
    blur_surface.fill((0, 0, 0, 128))
    screen.blit(blur_surface, (0, 0))

    ring_surface = pygame.Surface(screen.get_size(), pygame.SRCALPHA)
    center = (screen.get_width() // 2, screen.get_height() // 2)
    outer_radius = screen.get_width() // 2
    inner_radius = outer_radius - blue_ring_thickness

    for i in range(outer_radius, inner_radius, -1):
        alpha = int(128 * (i - inner_radius) / blue_ring_thickness)
        pygame.draw.circle(ring_surface, (173, 216, 230, alpha), center, i)

    screen.blit(ring_surface, (0, 0))

    font = pygame.font.Font(None, 36)
    text_surface = font.render(text, True, (255, 255, 255))
    text_rect = text_surface.get_rect(center=center)
    screen.blit(text_surface, text_rect)
    pygame.display.update()


def run_app(app_index, screen):
    try:
        app_module_name = f'apps.app_{app_index}.app_{app_index}'
        app_module = importlib.import_module(app_module_name)
        app_module.run(screen)
    except ModuleNotFoundError:
        print(f"App {app_index} not found.")
    except Exception as e:
        print(f"Error opening app {app_index}: {e}")


def main():
    pygame.init()
    screen = pygame.display.set_mode((1080, 1080))
    pygame.display.set_caption("Voice Assistant Home Screen")

    screen_size = screen.get_size()
    background = load_background('./resources/background.jpg', screen_size)
    circles = create_circles(screen_size)

    recorder = AudioToTextRecorder(spinner=False, model="tiny.en", language="en",
                                   post_speech_silence_duration=0.1, silero_sensitivity=0.4)
    recorder.start()

    hot_words = ["jarvis", "alexa"]
    clock = pygame.time.Clock()
    running = True

    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                for circle in circles:
                    if circle.is_clicked(event.pos):
                        app_index = circle.app_index
                        run_app(app_index, screen)
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    running = False

        # Process voice input in the main loop
        query = recorder.text()
        if any(hot_word in query.lower() for hot_word in hot_words):
            print(f"User: {query}")
            apply_blur_ring_and_text(screen, query)

            recorder.stop()
            response = assist.ask_question_memory(query)
            print(f"Assistant: {response}")

            if "#" in response:
                parts = response.split("#")
                speech, command = parts[0], parts[1]
                if "open app" in command:
                    try:
                        app_index = int(command.split()[-1])
                        run_app(app_index, screen)
                    except ValueError:
                        print("Invalid app index in command.")
            else:
                speech = response

            assist.TTS(speech)
            recorder.start()

        draw_home_screen(screen, background, circles)
        pygame.display.flip()
        clock.tick(60)

    recorder.stop()
    pygame.quit()
    sys.exit()


if __name__ == '__main__':
    main()
