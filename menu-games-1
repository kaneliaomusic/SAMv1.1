#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include <ESPmDNS.h>
#include <WiFiUdp.h>
#include <ArduinoOTA.h>

// WiFi credentials (replace with your own)
const char* ssid = "...";
const char* password = "...";

// OLED display settings
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Pin definitions
const int BUZZER_PIN = 4;    // Passive buzzer on GPIO 4
const int UP_BUTTON = 16;    // Button 1 on GPIO 16 (Up/Guess A)
const int DOWN_BUTTON = 15;  // Button 2 on GPIO 15 (Down/Guess B/Select)
const int LED_PIN = 2;       // Internal LED

// Menu variables
int menuIndex = 0;
const int MENU_ITEMS = 3;    // Pong, Tone Guess, Toggle LED
bool inMenu = true;
unsigned long lastButtonPress = 0;
const int DEBOUNCE_DELAY = 200;

// Game states
enum GameState { MENU, PONG, TONE_GUESS, TOGGLE_LED };
GameState currentState = MENU;

// Pong variables
const unsigned long PADDLE_RATE = 64;
const unsigned long BALL_RATE = 16;
const uint8_t PADDLE_HEIGHT = 12;
const uint8_t SCORE_LIMIT = 9;
bool pong_game_over, pong_win;
uint8_t player_score, mcu_score;
uint8_t ball_x = 53, ball_y = 26;
uint8_t ball_dir_x = 1, ball_dir_y = 1;
unsigned long ball_update;
unsigned long paddle_update;
const uint8_t MCU_X = 12;
uint8_t mcu_y = 16;
const uint8_t PLAYER_X = 115;
uint8_t player_y = 16;

// Tone Guess variables
int round_number = 0;
int correct_answers = 0;
bool tone_game_over = false;
bool waiting_for_guess = false;
int current_freq = 0;
int option_a = 0;
int option_b = 0;
const int frequencies[] = {100, 200, 300, 400, 500};

void setup() {
  // Initialize pins
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(UP_BUTTON, INPUT_PULLUP);
  pinMode(DOWN_BUTTON, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);

  // Initialize random seed
  randomSeed(analogRead(0));

  // Initialize display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    for(;;); // Loop forever if display fails
  }

  // WiFi and OTA setup
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED) {
    delay(500);
    ESP.restart();
  }

  ArduinoOTA
    .onStart([]() {
      display.clearDisplay();
      display.setCursor(0, 0);
      display.println("OTA Update Started");
      display.display();
      noTone(BUZZER_PIN);
    })
    .onEnd([]() {
      display.println("OTA Update Complete!");
      display.display();
    })
    .onProgress([](unsigned int progress, unsigned int total) {
      display.clearDisplay();
      display.setCursor(0, 0);
      display.printf("Progress: %u%%\n", (progress / (total / 100)));
      display.display();
    })
    .onError([](ota_error_t error) {
      display.printf("Error[%u]: ", error);
      display.display();
    });

  ArduinoOTA.begin();

  // Initial display
  updateMenuDisplay();
  ball_update = millis();
  paddle_update = ball_update;
}

void loop() {
  ArduinoOTA.handle(); // Handle OTA updates at all times

  switch (currentState) {
    case MENU:
      handleMenu();
      break;
    case PONG:
      playPong();
      break;
    case TONE_GUESS:
      playToneGuess();
      break;
    case TOGGLE_LED:
      toggleLED();
      break;
  }
}

void handleMenu() {
  bool button1Pressed = (digitalRead(UP_BUTTON) == LOW);
  bool button2Pressed = (digitalRead(DOWN_BUTTON) == LOW);
  unsigned long currentTime = millis();

  if (button1Pressed && (currentTime - lastButtonPress > DEBOUNCE_DELAY)) {
    menuIndex = (menuIndex + 1) % MENU_ITEMS;
    lastButtonPress = currentTime;
    updateMenuDisplay();
    tone(BUZZER_PIN, 1000, 100); // Navigation feedback
  }

  if (button2Pressed && (currentTime - lastButtonPress > DEBOUNCE_DELAY)) {
    lastButtonPress = currentTime;
    tone(BUZZER_PIN, 1500, 100); // Selection feedback
    if (menuIndex == 0) {
      currentState = PONG;
      display.clearDisplay();
      drawCourt();
      display.display();
      pong_game_over = false;
      player_score = 0;
      mcu_score = 0;
    } else if (menuIndex == 1) {
      currentState = TONE_GUESS;
      resetToneGame();
      startNewRound();
    } else if (menuIndex == 2) {
      currentState = TOGGLE_LED;
    }
  }
}

void updateMenuDisplay() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("=== GAMES ===");
  display.println(menuIndex == 0 ? "> Pong" : "  Pong");
  display.println(menuIndex == 1 ? "> Tone Guess" : "  Tone Guess");
  display.println(menuIndex == 2 ? "> Toggle LED" : "  Toggle LED");
  display.display();
}

// Pong functions
void playPong() {
  bool update_needed = false;
  unsigned long time = millis();

  static bool up_state = false;
  static bool down_state = false;
  
  up_state |= (digitalRead(UP_BUTTON) == LOW);
  down_state |= (digitalRead(DOWN_BUTTON) == LOW);

  if (time > ball_update) {
    uint8_t new_x = ball_x + ball_dir_x;
    uint8_t new_y = ball_y + ball_dir_y;

    if (new_x == 0 || new_x == 127) {
      ball_dir_x = -ball_dir_x;
      new_x += ball_dir_x + ball_dir_x;
      digitalWrite(LED_PIN, HIGH);
      if (new_x < 64) {
        player_scoreTone();
        player_score++;
      } else {
        mcu_scoreTone();
        mcu_score++;
      }
      digitalWrite(LED_PIN, LOW);

      if (player_score == SCORE_LIMIT || mcu_score == SCORE_LIMIT) {
        pong_win = player_score > mcu_score;
        pong_game_over = true;
      }
    }

    if (new_y == 0 || new_y == 53) {
      wallTone();
      ball_dir_y = -ball_dir_y;
      new_y += ball_dir_y + ball_dir_y;
    }

    if (new_x == MCU_X && new_y >= mcu_y && new_y <= mcu_y + PADDLE_HEIGHT) {
      mcuPaddleTone();
      ball_dir_x = -ball_dir_x;
      new_x += ball_dir_x + ball_dir_x;
    }

    if (new_x == PLAYER_X && new_y >= player_y && new_y <= player_y + PADDLE_HEIGHT) {
      playerPaddleTone();
      ball_dir_x = -ball_dir_x;
      new_x += ball_dir_x + ball_dir_x;
    }

    display.drawPixel(ball_x, ball_y, SSD1306_BLACK);
    display.drawPixel(new_x, new_y, SSD1306_WHITE);
    ball_x = new_x;
    ball_y = new_y;
    ball_update += BALL_RATE;
    update_needed = true;
  }

  if (time > paddle_update) {
    paddle_update += PADDLE_RATE;
    display.drawFastVLine(MCU_X, mcu_y, PADDLE_HEIGHT, SSD1306_BLACK);
    const uint8_t half_paddle = PADDLE_HEIGHT >> 1;
    if (mcu_y + half_paddle > ball_y) {
      int8_t dir = ball_x > MCU_X ? -1 : 1;
      mcu_y += dir;
    }
    if (mcu_y + half_paddle < ball_y) {
      int8_t dir = ball_x > MCU_X ? 1 : -1;
      mcu_y += dir;
    }
    if (mcu_y < 1) mcu_y = 1;
    if (mcu_y + PADDLE_HEIGHT > 53) mcu_y = 53 - PADDLE_HEIGHT;

    display.drawFastVLine(MCU_X, mcu_y, PADDLE_HEIGHT, SSD1306_WHITE);
    display.drawFastVLine(PLAYER_X, player_y, PADDLE_HEIGHT, SSD1306_BLACK);

    if (up_state) player_y -= 1;
    if (down_state) player_y += 1;
    up_state = down_state = false;

    if (player_y < 1) player_y = 1;
    if (player_y + PADDLE_HEIGHT > 53) player_y = 53 - PADDLE_HEIGHT;

    display.drawFastVLine(PLAYER_X, player_y, PADDLE_HEIGHT, SSD1306_WHITE);
    update_needed = true;
  }

  if (update_needed) {
    if (pong_game_over) {
      const char* text = pong_win ? "YOU WIN!!" : "YOU LOSE!";
      display.clearDisplay();
      display.setCursor(40, 28);
      display.print(text);
      display.display();
      delay(5000);

      currentState = MENU;
      ball_x = 53; ball_y = 26;
      ball_dir_x = 1; ball_dir_y = 1;
      mcu_y = 16; player_y = 16;
      mcu_score = 0; player_score = 0;
      pong_game_over = false;
      updateMenuDisplay();
      return;
    }

    display.setTextColor(SSD1306_WHITE, SSD1306_BLACK);
    display.setCursor(0, 56);
    display.print(mcu_score);
    display.setCursor(122, 56);
    display.print(player_score);
    display.display();
  }
}

// Tone Guess functions
void playToneGuess() {
  if (tone_game_over) {
    showResults();
    delay(5000);
    currentState = MENU;
    updateMenuDisplay();
    return;
  }

  if (waiting_for_guess) {
    if (digitalRead(UP_BUTTON) == LOW) {
      checkGuess(option_a);
      while (digitalRead(UP_BUTTON) == LOW) delay(10);
      waiting_for_guess = false;
      startNewRound();
    } else if (digitalRead(DOWN_BUTTON) == LOW) {
      checkGuess(option_b);
      while (digitalRead(DOWN_BUTTON) == LOW) delay(10);
      waiting_for_guess = false;
      startNewRound();
    }
  }
}

void startNewRound() {
  if (round_number >= 5) {
    tone_game_over = true;
    return;
  }

  round_number++;
  digitalWrite(LED_PIN, HIGH);

  current_freq = frequencies[random(0, 5)];
  option_a = current_freq;
  option_b = frequencies[random(0, 5)];
  while (option_b == option_a) {
    option_b = frequencies[random(0, 5)];
  }

  tone(BUZZER_PIN, current_freq, 500);
  delay(600);
  digitalWrite(LED_PIN, LOW);

  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("Round ");
  display.println(round_number);
  display.setCursor(0, 20);
  display.print("A: ");
  display.print(option_a);
  display.println(" Hz (UP)");
  display.setCursor(0, 40);
  display.print("B: ");
  display.print(option_b);
  display.println(" Hz (DOWN)");
  display.display();

  waiting_for_guess = true;
}

void checkGuess(int guess) {
  display.clearDisplay();
  display.setCursor(0, 20);
  if (guess == current_freq) {
    display.println("Correct!");
    correct_answers++;
    tone(BUZZER_PIN, 500, 200);
  } else {
    display.print("Wrong! It was ");
    display.print(current_freq);
    display.println(" Hz");
    tone(BUZZER_PIN, 200, 200);
  }
  display.display();
  delay(2000);
}

void showResults() {
  display.clearDisplay();
  display.setCursor(0, 20);
  display.print("Game Over! Score: ");
  display.print(correct_answers);
  display.println("/5");
  display.display();
}

void resetToneGame() {
  round_number = 0;
  correct_answers = 0;
  tone_game_over = false;
  waiting_for_guess = false;
}

// Toggle LED function
void toggleLED() {
  digitalWrite(LED_PIN, !digitalRead(LED_PIN)); // Toggle LED state
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 20);
  display.print("LED: ");
  display.println(digitalRead(LED_PIN) ? "ON" : "OFF");
  display.display();
  delay(1000); // Show state for 1 second
  currentState = MENU;
  updateMenuDisplay();
}

// Pong tone and court functions
void playerPaddleTone() {
  tone(BUZZER_PIN, 250, 25);
  delay(25);
}

void mcuPaddleTone() {
  tone(BUZZER_PIN, 225, 25);
  delay(25);
}

void wallTone() {
  tone(BUZZER_PIN, 200, 25);
  delay(25);
}

void player_scoreTone() {
  tone(BUZZER_PIN, 200, 25);
  delay(50);
  tone(BUZZER_PIN, 250, 25);
  delay(25);
}

void mcu_scoreTone() {
  tone(BUZZER_PIN, 250, 25);
  delay(25);
  tone(BUZZER_PIN, 200, 25);
  delay(25);
}

void drawCourt() {
  display.drawRect(0, 0, 128, 54, SSD1306_WHITE);
}
