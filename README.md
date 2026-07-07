# C-EGE----
基于C语言和EGE图形库设计的贪吃蛇小游戏
#include <graphics.h>
#include <stdio.h>
#include <time.h>
#include <string.h>
#include <math.h>
#include <algorithm>
#include <windows.h>
#include <mmsystem.h>
#pragma comment(lib, "winmm.lib")

#define NODE_WIDTH 40
#define MAX_OBSTACLES 20

using namespace std;

// 节点
typedef struct {
    int x;
    int y;
} node;

// 方向枚举
enum Direction {
    DIR_UP,
    DIR_DOWN,
    DIR_LEFT,
    DIR_RIGHT
};

// 游戏模式
enum GameMode {
    MODE_CLASSIC,    // 经典模式
    MODE_TIMED,      // 限时模式
    MODE_OBSTACLE    // 障碍模式
};

// 皮肤颜色
enum SkinColor {
    SKIN_SUNSET,     // 日落橙
    SKIN_CORAL,      // 珊瑚粉
    SKIN_GOLD,       // 金色
    SKIN_AMBER       // 琥珀色
};

// 背景主题
enum BackgroundTheme {
    THEME_FOREST,    // 森林主题
    THEME_DESERT,    // 沙漠主题
    THEME_OCEAN,     // 海洋主题
    THEME_SPACE      // 太空主题
};

// 游戏状态
int score = 0;
int highScore = 0;
int timeLeft = 120; // 限时模式的时间（秒）
bool gamePaused = false;
enum GameMode currentMode = MODE_CLASSIC;
enum SkinColor currentSkin = SKIN_SUNSET;
enum BackgroundTheme currentTheme = THEME_SPACE; // 默认主题改为太空
Direction currentDirection = DIR_RIGHT;

// 游戏设置
int gameSpeed = 8; // 默认速度 (1-20)
int soundVolume = 80; // 默认音量 (0-100)
int musicVolume = 70; // 默认音乐音量 (0-100)

// 障碍物
node obstacles[MAX_OBSTACLES];
int obstacleCount = 0;

// 音效控制
bool soundEnabled = true;
bool musicEnabled = true;

// 音乐控制
bool menuMusicPlaying = false;
bool gameMusicPlaying = false;

// 颜色函数 - 暖色调方案
color_t getSkinColor() {
    switch(currentSkin) {
        case SKIN_SUNSET: return EGERGB(255, 140, 0);   // 日落橙
        case SKIN_CORAL: return EGERGB(255, 114, 111);  // 珊瑚粉
        case SKIN_GOLD: return EGERGB(255, 204, 51);    // 金色
        case SKIN_AMBER: return EGERGB(255, 164, 32);   // 琥珀色
        default: return EGERGB(255, 140, 0);
    }
}

// 从颜色值中提取RGB分量
void getRGB(color_t color, int &r, int &g, int &b) {
    r = (color >> 16) & 0xFF;
    g = (color >> 8) & 0xFF;
    b = color & 0xFF;
}

// 获取主题文字颜色
color_t getTextColor() {
    switch(currentTheme) {
        case THEME_SPACE:    return EGERGB(255, 255, 255); // 太空主题用白色文字
        case THEME_FOREST:   return EGERGB(30, 80, 30);    // 森林主题用深绿色文字
        case THEME_DESERT:   return EGERGB(120, 60, 10);   // 沙漠主题用棕色文字
        case THEME_OCEAN:    return EGERGB(20, 50, 120);   // 海洋主题用深蓝色文字
        default:             return EGERGB(0, 0, 0);       // 默认黑色
    }
}

// 获取高亮文字颜色
color_t getHighlightColor() {
    switch(currentTheme) {
        case THEME_SPACE:    return EGERGB(255, 200, 50);  // 太空主题用金色高亮
        case THEME_FOREST:   return EGERGB(255, 140, 0);   // 森林主题用橙色高亮
        case THEME_DESERT:   return EGERGB(255, 215, 0);   // 沙漠主题用金色高亮
        case THEME_OCEAN:    return EGERGB(0, 255, 255);   // 海洋主题用青色高亮
        default:             return EGERGB(255, 140, 0);   // 默认橙色
    }
}

// 播放音效函数
void playSound(const char* soundType) {
    if (!soundEnabled) return;

    if (strcmp(soundType, "eat") == 0) {
        // 播放吃食物音效
        static DWORD lastEatTime = 0;
        if (GetTickCount() - lastEatTime > 200) { // 防止音效重叠
            Beep(800, 100); // 简单的蜂鸣声模拟吃食物音效
            lastEatTime = GetTickCount();
        }
    }
    else if (strcmp(soundType, "death") == 0) {
        // 播放死亡音效
        Beep(200, 500); // 低音蜂鸣声模拟死亡音效
    }
    else if (strcmp(soundType, "menu") == 0) {
        // 菜单选择音效
        Beep(500, 100);
    }
}

// 播放菜单音乐
void playMenuMusic() {
    if (!musicEnabled) return;

    if (!menuMusicPlaying) {
        // 停止游戏音乐
        mciSendString("close game_music", NULL, 0, NULL);
        gameMusicPlaying = false;

        // 播放菜单音乐
        mciSendString("open \"D:\\界面音乐.mp3\" type mpegvideo alias menu_music", NULL, 0, NULL);
        char volumeCommand[50];
        sprintf(volumeCommand, "setaudio menu_music volume to %d", musicVolume * 10);
        mciSendString(volumeCommand, NULL, 0, NULL);
        mciSendString("play menu_music repeat", NULL, 0, NULL);
        menuMusicPlaying = true;
    }
}

// 播放游戏音乐
void playGameMusic() {
    if (!musicEnabled) return;

    if (!gameMusicPlaying) {
        // 停止菜单音乐
        mciSendString("close menu_music", NULL, 0, NULL);
        menuMusicPlaying = false;

        // 播放游戏音乐
        mciSendString("open \"D:\\游戏音乐.mp3\" type mpegvideo alias game_music", NULL, 0, NULL);
        char volumeCommand[50];
        sprintf(volumeCommand, "setaudio game_music volume to %d", musicVolume * 10);
        mciSendString(volumeCommand, NULL, 0, NULL);
        mciSendString("play game_music repeat", NULL, 0, NULL);
        gameMusicPlaying = true;
    }
}

// 停止所有音乐
void stopAllMusic() {
    mciSendString("close menu_music", NULL, 0, NULL);
    mciSendString("close game_music", NULL, 0, NULL);
    menuMusicPlaying = false;
    gameMusicPlaying = false;
}

// 更新音乐音量
void updateMusicVolume() {
    if (menuMusicPlaying) {
        char volumeCommand[50];
        sprintf(volumeCommand, "setaudio menu_music volume to %d", musicVolume * 10);
        mciSendString(volumeCommand, NULL, 0, NULL);
    }
    if (gameMusicPlaying) {
        char volumeCommand[50];
        sprintf(volumeCommand, "setaudio game_music volume to %d", musicVolume * 10);
        mciSendString(volumeCommand, NULL, 0, NULL);
    }
}

// 绘制主题背景
void paintBackground() {
    switch(currentTheme) {
        case THEME_FOREST:
            // 森林主题 - 绿色调
            for (int y = 0; y < 600; y++) {
                int r = 100 - y / 600.0 * 30;
                int g = 180 - y / 600.0 * 40;
                int b = 100 - y / 600.0 * 30;
                setcolor(EGERGB(r, g, b));
                line(0, y, 800, y);
            }
            break;

        case THEME_DESERT:
            // 沙漠主题 - 橙黄色调
            for (int y = 0; y < 600; y++) {
                int r = 240 - y / 600.0 * 40;
                int g = 220 - y / 600.0 * 50;
                int b = 100 - y / 600.0 * 40;
                setcolor(EGERGB(r, g, b));
                line(0, y, 800, y);
            }
            break;

        case THEME_OCEAN:
            // 海洋主题 - 蓝色调
            for (int y = 0; y < 600; y++) {
                int r = 50 - y / 600.0 * 20;
                int g = 120 - y / 600.0 * 40;
                int b = 200 - y / 600.0 * 50;
                setcolor(EGERGB(r, g, b));
                line(0, y, 800, y);
            }
            break;

        case THEME_SPACE:
            // 太空主题 - 深色调带星星
            setbkcolor(EGERGB(10, 10, 40));
            cleardevice();

            // 绘制星星
            setcolor(EGERGB(255, 255, 255));
            for (int i = 0; i < 100; i++) {
                int x = random(800);
                int y = random(600);
                int size = random(2) + 1;
                setfillcolor(EGERGB(255, 255, 255));
                fillellipse(x, y, size, size);
            }
            return;
    }

    // 绘制装饰性花纹
    setcolor(EGERGB(255, 200, 150));
    setlinestyle(PS_SOLID, 1);
    for (int i = 0; i < 20; i++) {
        int x = random(800);
        int y = random(600);
        int size = random(10) + 5;
        circle(x, y, size);
    }
}

// 绘制艺术化网格
void paintGrid() {
    switch(currentTheme) {
        case THEME_FOREST:
            setcolor(EGERGB(120, 180, 120));
            break;
        case THEME_DESERT:
            setcolor(EGERGB(220, 180, 120));
            break;
        case THEME_OCEAN:
            setcolor(EGERGB(100, 160, 220));
            break;
        case THEME_SPACE:
            setcolor(EGERGB(80, 80, 160));
            break;
    }

    setlinestyle(PS_DASH, 1);

    // 横线
    for (int y = 0; y < 600; y += NODE_WIDTH) {
        line(0, y, 800, y);
    }
    // 竖线
    for (int x = 0; x < 800; x += NODE_WIDTH) {
        line(x, 0, x, 600);
    }

    setlinestyle(PS_SOLID, 1);
}

// 绘制蛇 - 艺术化设计
void paintSnake(node* snake, int n) {
    color_t snakeColor = getSkinColor();

    // 提取RGB分量
    int r, g, b;
    getRGB(snakeColor, r, g, b);

    for (int i = 0; i < n; i++) {
        int centerX = snake[i].x * NODE_WIDTH + NODE_WIDTH/2;
        int centerY = snake[i].y * NODE_WIDTH + NODE_WIDTH/2;

        // 蛇身使用圆形设计，更具艺术感
        if (i == 0) {
            // 蛇头 - 特殊设计
            setfillcolor(EGERGB(255, 240, 200)); // 米白色蛇头
            fillellipse(centerX, centerY, NODE_WIDTH/2, NODE_WIDTH/2);

            // 添加眼睛
            setfillcolor(EGERGB(50, 50, 50));
            if (currentDirection == DIR_RIGHT) {
                fillellipse(centerX + 5, centerY - 5, 3, 3);
                fillellipse(centerX + 5, centerY + 5, 3, 3);
            } else if (currentDirection == DIR_LEFT) {
                fillellipse(centerX - 5, centerY - 5, 3, 3);
                fillellipse(centerX - 5, centerY + 5, 3, 3);
            } else if (currentDirection == DIR_UP) {
                fillellipse(centerX - 5, centerY - 5, 3, 3);
                fillellipse(centerX + 5, centerY - 5, 3, 3);
            } else {
                fillellipse(centerX - 5, centerY + 5, 3, 3);
                fillellipse(centerX + 5, centerY + 5, 3, 3);
            }
        } else {
            // 蛇身 - 使用渐变色
            int colorFactor = 100 + (i * 155 / n);
            int newR = min(255, r - 30 + colorFactor/10);
            int newG = min(255, g - 30 + colorFactor/10);
            int newB = min(255, b - 30 + colorFactor/10);

            setfillcolor(EGERGB(newR, newG, newB));
            fillellipse(centerX, centerY, NODE_WIDTH/2 - 2, NODE_WIDTH/2 - 2);
        }
    }
}

// 绘制艺术化障碍物
void paintObstacles() {
    if (currentMode != MODE_OBSTACLE) return;

    setfillcolor(EGERGB(160, 100, 80)); // 暖棕色障碍物

    for (int i = 0; i < obstacleCount; i++) {
        int left = obstacles[i].x * NODE_WIDTH;
        int top = obstacles[i].y * NODE_WIDTH;
        int right = (obstacles[i].x + 1) * NODE_WIDTH;
        int bottom = (obstacles[i].y + 1) * NODE_WIDTH;

        // 绘制圆角矩形障碍物
        int radius = 8;
        bar(left + radius, top, right - radius, bottom);
        bar(left, top + radius, right, bottom - radius);
        fillellipse(left + radius, top + radius, radius, radius);
        fillellipse(right - radius, top + radius, radius, radius);
        fillellipse(left + radius, bottom - radius, radius, radius);
        fillellipse(right - radius, bottom - radius, radius, radius);
    }
}

// 蛇身体移动
node snakeMove(node* snake, int length, enum Direction direction) {
    currentDirection = direction; // 更新当前方向
    node tail = snake[length - 1];

    for (int i = length - 1; i > 0; i--) {
        snake[i] = snake[i - 1];
    }

    node newHead = snake[0];
    switch (direction) {
        case DIR_UP: newHead.y--; break;
        case DIR_DOWN: newHead.y++; break;
        case DIR_LEFT: newHead.x--; break;
        case DIR_RIGHT: newHead.x++; break;
    }

    snake[0] = newHead;
    return tail;
}

// 键盘输入改变direction
void changeDirection(enum Direction* pD) {
    if (kbhit()) {
        int key = getch();

        // 使用EGE的键盘常量
        if (key == key_up || key == 'w' || key == 'W') {
            if (*pD != DIR_DOWN) *pD = DIR_UP;
        } else if (key == key_down || key == 's' || key == 'S') {
            if (*pD != DIR_UP) *pD = DIR_DOWN;
        } else if (key == key_left || key == 'a' || key == 'A') {
            if (*pD != DIR_RIGHT) *pD = DIR_LEFT;
        } else if (key == key_right || key == 'd' || key == 'D') {
            if (*pD != DIR_LEFT) *pD = DIR_RIGHT;
        } else if (key == 'p' || key == 'P') {
            gamePaused = !gamePaused; // 暂停游戏
            playSound("menu");
        }
    }
}

// 绘制艺术化食物
void paintFood(node food) {
    int centerX = food.x * NODE_WIDTH + NODE_WIDTH/2;
    int centerY = food.y * NODE_WIDTH + NODE_WIDTH/2;

    // 绘制苹果状食物
    setfillcolor(EGERGB(255, 60, 60)); // 红色苹果
    fillellipse(centerX, centerY, NODE_WIDTH/2 - 2, NODE_WIDTH/2 - 2);

    // 苹果柄
    setcolor(EGERGB(120, 70, 20));
    setlinestyle(PS_SOLID, 2);
    line(centerX, centerY - NODE_WIDTH/2 + 2, centerX, centerY - NODE_WIDTH/2 - 3);

    // 苹果叶
    setfillcolor(EGERGB(100, 220, 100));
    fillellipse(centerX - 2, centerY - NODE_WIDTH/2 - 3, 4, 3);

    setlinestyle(PS_SOLID, 1);
}

// 随机创建食物
node createFood(node* snake, int length) {
    node food;
    while (1) {
        food.x = random(800 / NODE_WIDTH);
        food.y = random(600 / NODE_WIDTH);

        bool collision = false;

        // 检查是否与蛇身碰撞
        for (int i = 0; i < length; i++) {
            if (snake[i].x == food.x && snake[i].y == food.y) {
                collision = true;
                break;
            }
        }

        // 检查是否与障碍物碰撞（障碍模式）
        if (currentMode == MODE_OBSTACLE) {
            for (int i = 0; i < obstacleCount; i++) {
                if (obstacles[i].x == food.x && obstacles[i].y == food.y) {
                    collision = true;
                    break;
                }
            }
        }

        if (!collision) break;
    }
    return food;
}

// 创建障碍物
void createObstacles(node* snake, int length, node food) {
    obstacleCount = 0;

    // 创建一些随机障碍物
    for (int i = 0; i < MAX_OBSTACLES / 2; i++) {
        node obs;
        bool valid = false;

        while (!valid && obstacleCount < MAX_OBSTACLES) {
            obs.x = random(800 / NODE_WIDTH);
            obs.y = random(600 / NODE_WIDTH);
            valid = true;

            // 检查是否与蛇身碰撞
            for (int j = 0; j < length; j++) {
                if (snake[j].x == obs.x && snake[j].y == obs.y) {
                    valid = false;
                    break;
                }
            }

            // 检查是否与食物碰撞
            if (food.x == obs.x && food.y == obs.y) {
                valid = false;
            }

            // 检查是否与其他障碍物碰撞
            for (int j = 0; j < obstacleCount; j++) {
                if (obstacles[j].x == obs.x && obstacles[j].y == obs.y) {
                    valid = false;
                    break;
                }
            }

            if (valid) {
                obstacles[obstacleCount++] = obs;
            }
        }
    }
}

// 检查游戏是否结束
bool isGameOver(node *snake, int length) {
    // 是否撞墙
    if (snake[0].x < 0 || snake[0].x >= 800 / NODE_WIDTH)
        return true;

    if (snake[0].y < 0 || snake[0].y >= 600 / NODE_WIDTH)
        return true;

    // 是否吃到蛇身
    for (int i = 1; i < length; i++) {
        if (snake[0].x == snake[i].x && snake[0].y == snake[i].y)
            return true;
    }

    // 障碍模式下检查是否撞到障碍物
    if (currentMode == MODE_OBSTACLE) {
        for (int i = 0; i < obstacleCount; i++) {
            if (snake[0].x == obstacles[i].x && snake[0].y == obstacles[i].y)
                return true;
        }
    }

    return false;
}

// 检查限时模式是否结束
bool isTimeUp() {
    return (currentMode == MODE_TIMED && timeLeft <= 0);
}

// 重置游戏
void reset(node* snake, int *pLength, enum Direction *d) {
    snake[0] = node{5, 7};
    snake[1] = node{4, 7};
    snake[2] = node{3, 7};
    snake[3] = node{2, 7};
    snake[4] = node{1, 7};
    *pLength = 5;
    *d = DIR_RIGHT;
    currentDirection = DIR_RIGHT;
    score = 0;

    // 重置限时模式的时间
    if (currentMode == MODE_TIMED) {
        timeLeft = 120; // 2分钟
    }
}

// 绘制艺术化游戏信息面板
void paintGameInfo() {
    // 设置文字颜色
    color_t textColor = getTextColor();
    setcolor(textColor);
    setfont(20, 0, "楷体");

    // 绘制分数信息
    char scoreText[50];
    sprintf(scoreText, "分数: %d", score);
    outtextxy(20, 20, scoreText);

    char highScoreText[50];
    sprintf(highScoreText, "最高分: %d", highScore);
    outtextxy(20, 45, highScoreText);

    // 显示当前模式
    const char* modeText;
    switch(currentMode) {
        case MODE_CLASSIC: modeText = "经典模式"; break;
        case MODE_TIMED: modeText = "限时模式"; break;
        case MODE_OBSTACLE: modeText = "障碍模式"; break;
        default: modeText = "未知模式";
    }
    outtextxy(600, 20, modeText);

    // 限时模式显示剩余时间
    if (currentMode == MODE_TIMED) {
        char timeText[50];
        sprintf(timeText, "时间: %02d:%02d", timeLeft / 60, timeLeft % 60);
        outtextxy(600, 45, timeText);
    }

    // 暂停提示
    if (gamePaused) {
        setcolor(getHighlightColor());
        setfont(36, 0, "楷体");
        outtextxy(300, 250, "游戏暂停");
        setfont(20, 0, "楷体");
        outtextxy(330, 300, "按 P 键继续");
    }
}

// 绘制艺术化开始菜单
int showMenu() {
    cleardevice();
    paintBackground();

    // 播放菜单音乐
    playMenuMusic();

    // 获取文字颜色
    color_t textColor = getTextColor();
    color_t highlightColor = getHighlightColor();

    // 绘制标题
    setcolor(highlightColor);
    setfont(48, 0, "华文行楷");
    outtextxy(280, 30, "贪吃蛇游戏");

    // 绘制装饰性边框
    setcolor(highlightColor);
    setlinestyle(PS_SOLID, 3);

    // 绘制更美观的装饰性边框
    int borderPadding = 20;
    int borderWidth = 760;
    int borderHeight = 520;
    int borderX = 20;
    int borderY = 80;

    // 圆角矩形边框
    int radius = 20;
    arc(borderX + radius, borderY + radius, 180, 270, radius);
    arc(borderX + borderWidth - radius, borderY + radius, 270, 360, radius);
    arc(borderX + borderWidth - radius, borderY + borderHeight - radius, 0, 90, radius);
    arc(borderX + radius, borderY + borderHeight - radius, 90, 180, radius);

    line(borderX + radius, borderY, borderX + borderWidth - radius, borderY);
    line(borderX + borderWidth, borderY + radius, borderX + borderWidth, borderY + borderHeight - radius);
    line(borderX + borderWidth - radius, borderY + borderHeight, borderX + radius, borderY + borderHeight);
    line(borderX, borderY + radius, borderX, borderY + borderHeight - radius);

    // 左列设置
    int leftColumnX = 150;
    int rightColumnX = 500;
    int sectionStartY = 120;
    int optionSpacing = 40;

    // 绘制模式选择
    setcolor(highlightColor);
    setfont(28, 0, "楷体");
    outtextxy(leftColumnX, sectionStartY, "选择模式");

    setcolor(textColor);
    setfont(22, 0, "楷体");
    outtextxy(leftColumnX, sectionStartY + optionSpacing, "1. 经典模式");
    outtextxy(leftColumnX, sectionStartY + optionSpacing * 2, "2. 限时模式");
    outtextxy(leftColumnX, sectionStartY + optionSpacing * 3, "3. 障碍模式");

    // 绘制皮肤选择
    int skinSectionY = sectionStartY + optionSpacing * 4 + 20;
    setcolor(highlightColor);
    setfont(28, 0, "楷体");
    outtextxy(leftColumnX, skinSectionY, "选择皮肤");

    setcolor(textColor);
    setfont(22, 0, "楷体");
    outtextxy(leftColumnX, skinSectionY + optionSpacing, "4. 日落橙");
    outtextxy(leftColumnX, skinSectionY + optionSpacing * 2, "5. 珊瑚粉");
    outtextxy(leftColumnX, skinSectionY + optionSpacing * 3, "6. 金色");
    outtextxy(leftColumnX, skinSectionY + optionSpacing * 4, "7. 琥珀色");

    // 绘制主题选择
    setcolor(highlightColor);
    setfont(28, 0, "楷体");
    outtextxy(rightColumnX, sectionStartY, "选择主题");

    setcolor(textColor);
    setfont(22, 0, "楷体");
    outtextxy(rightColumnX, sectionStartY + optionSpacing, "8. 森林");
    outtextxy(rightColumnX, sectionStartY + optionSpacing * 2, "9. 沙漠");
    outtextxy(rightColumnX, sectionStartY + optionSpacing * 3, "0. 海洋");
    outtextxy(rightColumnX, sectionStartY + optionSpacing * 4, "Q. 太空");

    // 绘制设置选项
    int settingsSectionY = sectionStartY + optionSpacing * 5 + 20;
    setcolor(highlightColor);
    setfont(28, 0, "楷体");
    outtextxy(rightColumnX, settingsSectionY, "其他选项");

    setcolor(textColor);
    setfont(22, 0, "楷体");
    outtextxy(rightColumnX, settingsSectionY + optionSpacing, "S. 游戏设置");
    outtextxy(rightColumnX, settingsSectionY + optionSpacing * 2, "H. 游戏帮助");

    // 开始游戏选项
    int startGameY = settingsSectionY + optionSpacing * 3 + 40;
    setcolor(highlightColor);
    setfont(28, 0, "楷体");
    outtextxy(320, startGameY, "按 Enter 开始游戏");

    // 绘制装饰性蛇形图案
    setcolor(highlightColor);
    setlinestyle(PS_SOLID, 4);

    // 左上角装饰
    arc(100, 130, 180, 270, 30);
    arc(100, 350, 90, 180, 30);

    // 右上角装饰
    arc(700, 130, 270, 360, 30);
    arc(700, 350, 0, 90, 30);

    // 等待用户选择
    while (true) {
        if (kbhit()) {
            int key = getch();
            if (key >= '1' && key <= '9' || key == '0' || key == 'q' ||
                key == 'Q' || key == 's' || key == 'S' || key == 'h' ||
                key == 'H' || key == 13) {
                playSound("menu");
                return key;
            }
        }
        delay_fps(30);
    }
}

// 显示游戏设置菜单
void showSettings() {
    int selected = 0; // 0: 速度, 1: 音量, 2: 音效开关, 3: 音乐开关

    while (true) {
        cleardevice();
        paintBackground();

        // 获取文字颜色
        color_t textColor = getTextColor();
        color_t highlightColor = getHighlightColor();

        setcolor(highlightColor);
        setfont(36, 0, "华文行楷");
        outtextxy(320, 50, "游戏设置");

        setcolor(textColor);
        setfont(22, 0, "楷体");

        // 显示当前设置
        outtextxy(300, 120, "当前游戏速度:");
        char speedText[20];
        sprintf(speedText, "%d", gameSpeed);
        outtextxy(500, 120, speedText);

        outtextxy(300, 170, "当前音效音量:");
        char soundText[20];
        sprintf(soundText, "%d", soundVolume);
        outtextxy(500, 170, soundText);

        outtextxy(300, 220, "当前音乐音量:");
        char musicText[20];
        sprintf(musicText, "%d", musicVolume);
        outtextxy(500, 220, musicText);

        outtextxy(300, 270, "音效状态:");
        outtextxy(500, 270, soundEnabled ? "开启" : "关闭");

        outtextxy(300, 320, "音乐状态:");
        outtextxy(500, 320, musicEnabled ? "开启" : "关闭");

        outtextxy(300, 370, "操作说明:");
        outtextxy(320, 400, "↑/↓: 调整数值");
        outtextxy(320, 430, "←/→: 切换选项");
        outtextxy(320, 460, "空格: 切换开关");
        outtextxy(320, 490, "ESC: 返回主菜单");

        // 高亮当前选中的选项
        setcolor(highlightColor);
        if (selected == 0) {
            outtextxy(500, 120, speedText);
        } else if (selected == 1) {
            outtextxy(500, 170, soundText);
        } else if (selected == 2) {
            outtextxy(500, 220, musicText);
        } else if (selected == 3) {
            outtextxy(500, 270, soundEnabled ? "开启" : "关闭");
        } else if (selected == 4) {
            outtextxy(500, 320, musicEnabled ? "开启" : "关闭");
        }

        // 处理键盘输入
        if (kbhit()) {
            int key = getch();

            if (key == key_up) {
                // 增加数值
                if (selected == 0 && gameSpeed < 20) {
                    gameSpeed++;
                    playSound("menu");
                } else if (selected == 1 && soundVolume < 100) {
                    soundVolume += 10;
                    playSound("menu");
                } else if (selected == 2 && musicVolume < 100) {
                    musicVolume += 10;
                    updateMusicVolume();
                    playSound("menu");
                }
            }
            else if (key == key_down) {
                // 减少数值
                if (selected == 0 && gameSpeed > 1) {
                    gameSpeed--;
                    playSound("menu");
                } else if (selected == 1 && soundVolume > 0) {
                    soundVolume -= 10;
                    playSound("menu");
                } else if (selected == 2 && musicVolume > 0) {
                    musicVolume -= 10;
                    updateMusicVolume();
                    playSound("menu");
                }
            }
            else if (key == key_left) {
                // 选择上一个选项
                selected = (selected + 4) % 5;
                playSound("menu");
            }
            else if (key == key_right) {
                // 选择下一个选项
                selected = (selected + 1) % 5;
                playSound("menu");
            }
            else if (key == ' ') {
                // 切换开关
                if (selected == 3) {
                    soundEnabled = !soundEnabled;
                    playSound("menu");
                } else if (selected == 4) {
                    musicEnabled = !musicEnabled;
                    if (musicEnabled) {
                        if (menuMusicPlaying) {
                            playMenuMusic();
                        } else if (gameMusicPlaying) {
                            playGameMusic();
                        }
                    } else {
                        stopAllMusic();
                    }
                    playSound("menu");
                }
            }
            else if (key == 27) { // ESC键
                break;
            }
        }

        delay_fps(30);
    }
}

// 显示游戏帮助
void showHelp() {
    cleardevice();
    paintBackground();

    // 获取文字颜色
    color_t textColor = getTextColor();
    color_t highlightColor = getHighlightColor();

    setcolor(highlightColor);
    setfont(36, 0, "华文行楷");
    outtextxy(320, 50, "游戏帮助");

    setcolor(textColor);
    setfont(20, 0, "楷体");

    outtextxy(100, 120, "游戏规则:");
    outtextxy(120, 150, "1. 控制蛇吃食物，每吃一个食物得10分");
    outtextxy(120, 180, "2. 蛇不能撞墙或自己的身体");
    outtextxy(120, 210, "3. 障碍模式下还要避开障碍物");
    outtextxy(120, 240, "4. 限时模式需要在时间结束前获得尽可能高分");

    outtextxy(100, 290, "操作说明:");
    outtextxy(120, 320, "方向键或WASD: 控制蛇的移动方向");
    outtextxy(120, 350, "P键: 暂停/继续游戏");
    outtextxy(120, 380, "ESC键: 退出游戏");

    outtextxy(100, 430, "模式说明:");
    outtextxy(120, 460, "经典模式: 传统贪吃蛇玩法");
    outtextxy(120, 490, "限时模式: 2分钟内获得最高分");
    outtextxy(120, 520, "障碍模式: 场景中有障碍物需要避开");

    setcolor(highlightColor);
    outtextxy(300, 560, "按任意键返回主菜单");

    while (!kbhit()) {
        delay_fps(30);
    }
    getch(); // 清除按键缓冲
    playSound("menu");
}

// 处理菜单选择
void handleMenuSelection(int selection) {
    switch(selection) {
        case '1': currentMode = MODE_CLASSIC; break;
        case '2': currentMode = MODE_TIMED; break;
        case '3': currentMode = MODE_OBSTACLE; break;
        case '4': currentSkin = SKIN_SUNSET; break;
        case '5': currentSkin = SKIN_CORAL; break;
        case '6': currentSkin = SKIN_GOLD; break;
        case '7': currentSkin = SKIN_AMBER; break;
        case '8': currentTheme = THEME_FOREST; break;
        case '9': currentTheme = THEME_DESERT; break;
        case '0': currentTheme = THEME_OCEAN; break;
        case 'q':
        case 'Q': currentTheme = THEME_SPACE; break;
        case 's':
        case 'S': showSettings(); break;
        case 'h':
        case 'H': showHelp(); break;
    }
}

// 显示艺术化游戏结束画面
bool showGameOver() {
    cleardevice();
    paintBackground();

    // 获取文字颜色
    color_t textColor = getTextColor();
    color_t highlightColor = getHighlightColor();

    setcolor(highlightColor);
    setfont(56, 0, "华文行楷");
    outtextxy(280, 150, "游戏结束");

    setcolor(textColor);
    setfont(28, 0, "楷体");

    char scoreText[50];
    sprintf(scoreText, "最终分数: %d", score);
    outtextxy(340, 230, scoreText);

    if (score > highScore) {
        setcolor(highlightColor);
        outtextxy(340, 270, "新高分!");
        highScore = score;
    }

    setcolor(textColor);
    outtextxy(300, 330, "按 R 键重新开始");
    outtextxy(300, 370, "按 M 键返回菜单");
    outtextxy(300, 410, "按 ESC 键退出");

    // 绘制装饰性图案
    setcolor(highlightColor);
    setlinestyle(PS_SOLID, 3);
    circle(200, 250, 40);
    circle(600, 250, 40);
    circle(200, 400, 40);
    circle(600, 400, 40);

    // 播放死亡音效
    playSound("death");

    // 等待用户选择
    while (true) {
        if (kbhit()) {
            int key = getch();
            if (key == 'r' || key == 'R') {
                playSound("menu");
                return true; // 重新开始
            } else if (key == 'm' || key == 'M') {
                playSound("menu");
                return false; // 返回菜单
            } else if (key == 27) { // ESC键的ASCII码是27
                closegraph();
                exit(0);
            }
        }
        delay_fps(30);
    }
}

int main() {
    // 初始化EGE图形库
    setinitmode(INIT_ANIMATION);
    initgraph(800, 600);
    randomize();

    // 游戏主循环
    while (true) {
        // 显示菜单
        int selection = showMenu();
        if (selection == 13) { // 回车键开始游戏
            // 初始化游戏
            setbkcolor(EGERGB(255, 240, 220)); // 暖米色背景
            cleardevice();

            // 播放游戏音乐
            playGameMusic();

            node snake[100] = { {5, 7}, {4, 7}, {3, 7}, {2, 7}, {1, 7} };
            int length = 5;
            enum Direction d = DIR_RIGHT;
            currentDirection = DIR_RIGHT;

            node food = createFood(snake, length);

            // 障碍模式下创建障碍物
            if (currentMode == MODE_OBSTACLE) {
                createObstacles(snake, length, food);
            }

            // 游戏循环
            time_t lastTime = time(NULL);
            bool restart = false;

            for (; is_run() && !restart; delay_fps(gameSpeed)) {
                // 处理暂停
                if (gamePaused) {
                    paintBackground();
                    paintGrid();
                    paintObstacles();
                    paintSnake(snake, length);
                    paintFood(food);
                    paintGameInfo();
                    continue;
                }

                // 更新限时模式的时间
                if (currentMode == MODE_TIMED) {
                    time_t currentTime = time(NULL);
                    if (currentTime != lastTime) {
                        timeLeft--;
                        lastTime = currentTime;
                    }
                }

                // 绘制游戏元素
                paintBackground();
                paintGrid();
                paintObstacles();
                paintSnake(snake, length);
                paintFood(food);
                paintGameInfo();

                changeDirection(&d);
                node lastTail = snakeMove(snake, length, d);

                // 检查是否吃到食物
                if (snake[0].x == food.x && snake[0].y == food.y) {
                    if (length < 100) {
                        snake[length] = lastTail;
                        length++;
                    }
                    score += 10;
                    playSound("eat"); // 播放吃食物音效
                    food = createFood(snake, length);
                }

                // 检查游戏是否结束
                if (isGameOver(snake, length) || isTimeUp()) {
                    // 停止游戏音乐
                    mciSendString("close game_music", NULL, 0, NULL);
                    gameMusicPlaying = false;

                    restart = showGameOver();
                    if (restart) {
                        // 重新播放游戏音乐
                        playGameMusic();

                        reset(snake, &length, &d);
                        currentDirection = d;
                        if (currentMode == MODE_OBSTACLE) {
                            createObstacles(snake, length, food);
                        }
                        food = createFood(snake, length);
                    }
                    break;
                }
            }
        } else {
            handleMenuSelection(selection);
        }
    }

    // 程序退出前停止所有音乐
    stopAllMusic();
    closegraph();
    return 0;
}
