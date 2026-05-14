import os
import sys
import time
import threading
from enum import Enum, auto

import pygame
import qrcode
from PIL import Image, ImageDraw, ImageFont
import gphoto2 as gp


ASSETS_DIR = os.path.join(os.path.dirname(__file__), "assets")
CAPTURES_DIR = os.path.join(os.path.dirname(__file__), "captures")
SCREEN_SIZE = (1920, 1080)
FRAME_THUMBNAIL_SIZE = (420, 320)
COUNTDOWN_SECONDS = 3
PREVIEW_TIMEOUT_SECONDS = 15

FRAME_COLORS = [
    (242, 101, 34, 255),
    (55, 179, 255, 255),
    (61, 183, 107, 255),
    (169, 60, 199, 255),
]


class AppState(Enum):
    IDLE = auto()
    FRAME_SELECT = auto()
    COUNTDOWN = auto()
    PROCESSING = auto()
    PREVIEW = auto()
    ERROR = auto()


class GPhoto2Camera:
    def __init__(self):
        self.camera = gp.Camera()
        self.camera.init()

    def capture(self, target_path: str) -> str:
        file_path = self.camera.capture(gp.GP_CAPTURE_IMAGE)
        camera_file = self.camera.file_get(
            file_path.folder, file_path.name, gp.GP_FILE_TYPE_NORMAL
        )
        camera_file.save(target_path)
        try:
            self.camera.file_delete(file_path.folder, file_path.name)
        except gp.GPhoto2Error:
            pass
        return target_path

    def close(self):
        try:
            self.camera.exit()
        except Exception:
            pass


class PhotoboothApp:
    def __init__(self):
        pygame.init()
        self.screen = pygame.display.set_mode(SCREEN_SIZE, pygame.FULLSCREEN)
        pygame.display.set_caption("Touch Photobooth")
        self.clock = pygame.time.Clock()

        self.font_large = pygame.font.SysFont("arial", 120, bold=True)
        self.font_medium = pygame.font.SysFont("arial", 56)
        self.font_small = pygame.font.SysFont("arial", 34)

        self.state = AppState.IDLE
        self.selected_frame_index = None
        self.selected_frame_path = None
        self.final_image_path = None
        self.qr_image_path = None
        self.preview_text = ""
        self.error_message = ""
        self.state_start_time = 0
        self.countdown_start_ticks = 0
        self.processing_thread = None

        os.makedirs(ASSETS_DIR, exist_ok=True)
        os.makedirs(CAPTURES_DIR, exist_ok=True)

        self.ensure_frame_assets()
        self.frame_templates = self.load_frame_templates()
        self.camera = self.build_camera()

        self.button_done = pygame.Rect(SCREEN_SIZE[0] - 320, SCREEN_SIZE[1] - 140, 260, 90)
        self.button_retry = pygame.Rect(80, SCREEN_SIZE[1] - 140, 260, 90)

    def build_camera(self):
        try:
            return GPhoto2Camera()
        except Exception as exc:
            self.error_message = f"Camera init failed: {exc}"
            self.state = AppState.ERROR
            return None

    def ensure_frame_assets(self):
        for index, color in enumerate(FRAME_COLORS, start=1):
            path = os.path.join(ASSETS_DIR, f"frame{index}.png")
            if not os.path.exists(path):
                self.create_placeholder_frame(path, color, index)

    @staticmethod
    def create_placeholder_frame(path: str, accent_color, index: int):
        size = (1280, 960)
        frame = Image.new("RGBA", size, (0, 0, 0, 0))
        draw = ImageDraw.Draw(frame)

        border_thickness = 56
        draw.rectangle(
            [0, 0, size[0], size[1]],
            outline=accent_color,
            width=border_thickness,
        )

        inner = [border_thickness + 20, border_thickness + 20, size[0] - border_thickness - 20, size[1] - border_thickness - 20]
        draw.rectangle(inner, fill=(0, 0, 0, 0))

        corner = 120
        draw.rectangle([inner[0], inner[1], inner[0] + corner, inner[1] + 20], fill=accent_color)
        draw.rectangle([inner[0], inner[1], inner[0] + 20, inner[1] + corner], fill=accent_color)
        draw.rectangle([inner[2] - corner, inner[1], inner[2], inner[1] + 20], fill=accent_color)
        draw.rectangle([inner[2] - 20, inner[1], inner[2], inner[1] + corner], fill=accent_color)
        draw.rectangle([inner[0], inner[3] - 20, inner[0] + corner, inner[3]], fill=accent_color)
        draw.rectangle([inner[0], inner[3] - corner, inner[0] + 20, inner[3]], fill=accent_color)
        draw.rectangle([inner[2] - corner, inner[3] - 20, inner[2], inner[3]], fill=accent_color)
        draw.rectangle([inner[2] - 20, inner[3] - corner, inner[2], inner[3]], fill=accent_color)

        label = f"Frame {index}"
        font = ImageFont.load_default()
        text_size = draw.textsize(label, font=font)
        text_position = ((size[0] - text_size[0]) // 2, size[1] - text_size[1] - 40)
        draw.text(text_position, label, fill=accent_color, font=font)

        frame.save(path, "PNG")

    def load_frame_templates(self):
        templates = []
        for index in range(1, 5):
            path = os.path.join(ASSETS_DIR, f"frame{index}.png")
            pil_image = Image.open(path).convert("RGBA")
            thumbnail = pil_image.copy()
            thumbnail.thumbnail(FRAME_THUMBNAIL_SIZE, Image.LANCZOS)
            surface = pygame.image.fromstring(thumbnail.tobytes(), thumbnail.size, "RGBA")
            templates.append({"path": path, "surface": surface, "size": pil_image.size})
        return templates

    def run(self):
        running = True
        while running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                elif event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                    running = False
                elif event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                    self.handle_touch(event.pos)

            self.update_state()
            self.draw()
            pygame.display.flip()
            self.clock.tick(30)

        self.shutdown()

    def handle_touch(self, pos):
        if self.state == AppState.IDLE:
            self.state = AppState.FRAME_SELECT
        elif self.state == AppState.FRAME_SELECT:
            self.handle_frame_selection_touch(pos)
        elif self.state == AppState.PREVIEW:
            if self.button_done.collidepoint(pos):
                self.reset()
        elif self.state == AppState.ERROR:
            if self.button_retry.collidepoint(pos):
                self.reset()

    def handle_frame_selection_touch(self, pos):
        margin = 60
        x0 = margin
        y0 = 180
        spacing = 60
        current_x = x0
        current_y = y0
        max_x = SCREEN_SIZE[0] - FRAME_THUMBNAIL_SIZE[0] - margin

        for index, template in enumerate(self.frame_templates):
            rect = pygame.Rect(current_x, current_y, *FRAME_THUMBNAIL_SIZE)
            if rect.collidepoint(pos):
                self.selected_frame_index = index
                self.selected_frame_path = template["path"]
                self.start_countdown()
                return
            current_x += FRAME_THUMBNAIL_SIZE[0] + spacing
            if current_x > max_x:
                current_x = x0
                current_y += FRAME_THUMBNAIL_SIZE[1] + spacing

    def start_countdown(self):
        self.countdown_start_ticks = pygame.time.get_ticks()
        self.state = AppState.COUNTDOWN
        self.state_start_time = time.time()

    def update_state(self):
        if self.state == AppState.COUNTDOWN:
            elapsed = (pygame.time.get_ticks() - self.countdown_start_ticks) / 1000.0
            if elapsed >= COUNTDOWN_SECONDS:
                self.start_processing()
        elif self.state == AppState.PREVIEW:
            if time.time() - self.state_start_time >= PREVIEW_TIMEOUT_SECONDS:
                self.reset()
        elif self.state == AppState.PROCESSING and self.processing_thread is not None:
            if not self.processing_thread.is_alive():
                self.processing_thread = None
                if self.state != AppState.ERROR:
                    self.state_start_time = time.time()

    def start_processing(self):
        if self.selected_frame_path is None:
            self.error_message = "No frame selected."
            self.state = AppState.ERROR
            return

        self.state = AppState.PROCESSING
        self.processing_thread = threading.Thread(target=self.capture_and_process, daemon=True)
        self.processing_thread.start()

    def capture_and_process(self):
        try:
            if self.camera is None:
                raise RuntimeError("Camera unavailable")

            raw_path = os.path.join(CAPTURES_DIR, f"capture_{int(time.time())}.jpg")
            self.camera.capture(raw_path)

            framed_path = os.path.join(CAPTURES_DIR, f"framed_{int(time.time())}.png")
            self.final_image_path = self.overlay_photo_with_frame(raw_path, self.selected_frame_path, framed_path)

            qr_text = f"https://example.com/download/{os.path.basename(self.final_image_path)}"
            self.qr_image_path = os.path.join(CAPTURES_DIR, f"qr_{int(time.time())}.png")
            self.generate_qr(qr_text, self.qr_image_path)

            self.preview_text = "Scan to Download"
            self.state = AppState.PREVIEW
        except Exception as exc:
            self.error_message = f"Capture failed: {exc}"
            self.state = AppState.ERROR

    @staticmethod
    def resize_and_crop(photo: Image.Image, target_size):
        target_w, target_h = target_size
        source_w, source_h = photo.size
        source_ratio = source_w / source_h
        target_ratio = target_w / target_h

        if source_ratio > target_ratio:
            new_height = target_h
            new_width = int(source_ratio * target_h)
        else:
            new_width = target_w
            new_height = int(target_w / source_ratio)

        photo = photo.resize((new_width, new_height), Image.LANCZOS)
        left = (new_width - target_w) // 2
        top = (new_height - target_h) // 2
        return photo.crop((left, top, left + target_w, top + target_h))

    def overlay_photo_with_frame(self, photo_path: str, frame_path: str, output_path: str) -> str:
        frame = Image.open(frame_path).convert("RGBA")
        photo = Image.open(photo_path).convert("RGB")
        photo = self.resize_and_crop(photo, frame.size)

        background = Image.new("RGBA", frame.size)
        background.paste(photo, (0, 0))
        background.paste(frame, (0, 0), mask=frame)

        background.save(output_path, "PNG")
        return output_path

    @staticmethod
    def generate_qr(data: str, output_path: str):
        qr = qrcode.QRCode(
            version=2,
            error_correction=qrcode.constants.ERROR_CORRECT_H,
            box_size=10,
            border=4,
        )
        qr.add_data(data)
        qr.make(fit=True)
        img = qr.make_image(fill_color="black", back_color="white").convert("RGB")
        img = img.resize((520, 520), Image.LANCZOS)
        img.save(output_path, "PNG")

    def reset(self):
        self.state = AppState.IDLE
        self.selected_frame_index = None
        self.selected_frame_path = None
        self.final_image_path = None
        self.qr_image_path = None
        self.preview_text = ""
        self.error_message = ""
        self.state_start_time = 0
        self.countdown_start_ticks = 0
        self.processing_thread = None

    def draw(self):
        self.screen.fill((18, 18, 18))

        if self.state == AppState.IDLE:
            self.draw_idle()
        elif self.state == AppState.FRAME_SELECT:
            self.draw_frame_selection()
        elif self.state == AppState.COUNTDOWN:
            self.draw_countdown()
        elif self.state == AppState.PROCESSING:
            self.draw_processing()
        elif self.state == AppState.PREVIEW:
            self.draw_preview()
        elif self.state == AppState.ERROR:
            self.draw_error()

    def draw_idle(self):
        self.draw_centered_text("Touch to Start", self.font_large, (255, 255, 255), y_offset=-60)
        self.draw_centered_text("Choose a frame and capture with the DSLR", self.font_medium, (200, 200, 200), y_offset=80)

    def draw_frame_selection(self):
        self.draw_centered_text("Select a Frame", self.font_large, (255, 255, 255), y_offset=-420)
        margin = 60
        y = 180
        x = margin
        spacing = 60
        max_x = SCREEN_SIZE[0] - FRAME_THUMBNAIL_SIZE[0] - margin

        for index, template in enumerate(self.frame_templates):
            rect = pygame.Rect(x, y, *FRAME_THUMBNAIL_SIZE)
            pygame.draw.rect(self.screen, (80, 80, 80), rect, border_radius=24)
            self.screen.blit(template["surface"], rect.topleft)
            if index == self.selected_frame_index:
                pygame.draw.rect(self.screen, (255, 200, 0), rect, 10, border_radius=24)
            else:
                pygame.draw.rect(self.screen, (140, 140, 140), rect, 4, border_radius=24)

            label = self.font_small.render(f"Frame {index + 1}", True, (240, 240, 240))
            self.screen.blit(label, (x + 20, y + FRAME_THUMBNAIL_SIZE[1] + 10))
            x += FRAME_THUMBNAIL_SIZE[0] + spacing
            if x > max_x:
                x = margin
                y += FRAME_THUMBNAIL_SIZE[1] + 120

    def draw_countdown(self):
        elapsed = (pygame.time.get_ticks() - self.countdown_start_ticks) / 1000.0
        remaining = max(0, COUNTDOWN_SECONDS - int(elapsed))
        self.draw_centered_text("Get Ready!", self.font_medium, (255, 255, 255), y_offset=-120)
        self.draw_centered_text(str(remaining or 1), self.font_large, (255, 115, 35), y_offset=40)
        self.draw_centered_text("Touchscreen camera will fire shortly", self.font_small, (200, 200, 200), y_offset=220)

    def draw_processing(self):
        self.draw_centered_text("Capturing Photo", self.font_large, (255, 255, 255), y_offset=-40)
        self.draw_centered_text("Please wait while the DSLR saves the image", self.font_medium, (220, 220, 220), y_offset=120)

    def draw_preview(self):
        if self.final_image_path and os.path.exists(self.final_image_path):
            final_image = Image.open(self.final_image_path).convert("RGBA")
            final_image.thumbnail((1020, 760), Image.LANCZOS)
            image_surf = pygame.image.fromstring(final_image.tobytes(), final_image.size, "RGBA")
            self.screen.blit(image_surf, (80, 140))
            pygame.draw.rect(self.screen, (255, 255, 255), pygame.Rect(70, 130, final_image.size[0] + 20, final_image.size[1] + 20), 4, border_radius=16)

        if self.qr_image_path and os.path.exists(self.qr_image_path):
            qr_image = Image.open(self.qr_image_path).convert("RGB")
            qr_surf = pygame.image.fromstring(qr_image.tobytes(), qr_image.size, "RGB")
            qr_rect = qr_surf.get_rect(topright=(SCREEN_SIZE[0] - 80, 220))
            self.screen.blit(qr_surf, qr_rect)
            self.draw_text("Scan to Download", self.font_medium, (255, 255, 255), (qr_rect.left, qr_rect.bottom + 20))

        self.draw_text("Final framed photo", self.font_medium, (255, 255, 255), (80, 80))
        self.draw_text("QR code contains a download link placeholder.", self.font_small, (200, 200, 200), (80, 120))

        pygame.draw.rect(self.screen, (50, 160, 90), self.button_done, border_radius=20)
        self.draw_text("Done", self.font_medium, (255, 255, 255), self.button_done.center, center=True)

    def draw_error(self):
        self.draw_centered_text("Camera Error", self.font_large, (255, 100, 100), y_offset=-120)
        self.draw_centered_text(self.error_message, self.font_medium, (230, 230, 230), y_offset=20)
        pygame.draw.rect(self.screen, (200, 70, 70), self.button_retry, border_radius=20)
        self.draw_text("Try Again", self.font_medium, (255, 255, 255), self.button_retry.center, center=True)

    def draw_centered_text(self, text, font, color, y_offset=0):
        surface = font.render(text, True, color)
        rect = surface.get_rect(center=(SCREEN_SIZE[0] // 2, SCREEN_SIZE[1] // 2 + y_offset))
        self.screen.blit(surface, rect)

    def draw_text(self, text, font, color, position, center=False):
        surface = font.render(text, True, color)
        rect = surface.get_rect()
        if center:
            rect.center = position
        else:
            rect.topleft = position
        self.screen.blit(surface, rect)

    def shutdown(self):
        if self.camera:
            self.camera.close()
        pygame.quit()


if __name__ == "__main__":
    app = PhotoboothApp()
    app.run()
