#include "raylib.h"
#include <cmath>
#include <vector>
#include <cstdlib>

// ─── Constantes ───────────────────────────────────────────────
const int LARGURA_TELA       = 800;
const int ALTURA_TELA        = 600;
const float VELOCIDADE_GIRO  = 200.0f;
const float EMPUXO           = 300.0f;
const float ATRITO           = 0.98f;
const float VELOCIDADE_MAX   = 400.0f;

const int   TOTAL_ASTEROIDS  = 6;
const int   MAX_BALAS        = 8;
const float VELOCIDADE_BALA  = 550.0f;
const float TEMPO_VIDA_BALA  = 1.2f;   // segundos até sumir
const float COOLDOWN_TIRO    = 0.25f;  // tempo entre tiros

// ─── Estrutura da Nave ────────────────────────────────────────
struct Nave {
    Vector2 posicao;
    Vector2 velocidade;
    float   angulo;
    bool    acelerando;
    float   tempoCooldown;  // controla intervalo entre tiros
};

// ─── Estrutura da Bala ────────────────────────────────────────
struct Bala {
    Vector2 posicao;
    Vector2 velocidade;
    float   tempoVida;
    bool    ativa;
};

// ─── Tamanho do Asteroid ──────────────────────────────────────
enum TamanhoAsteroid { GRANDE = 0, MEDIO, PEQUENO };

// ─── Estrutura do Asteroid ────────────────────────────────────
struct Asteroid {
    Vector2         posicao;
    Vector2         velocidade;
    float           angulo;
    float           velocidadeRotacao;
    TamanhoAsteroid tamanho;
    bool            ativo;
};

// ─── Funções auxiliares ───────────────────────────────────────

void VerificarBordas(Vector2& posicao) {
    if (posicao.x < 0)            posicao.x = LARGURA_TELA;
    if (posicao.x > LARGURA_TELA) posicao.x = 0;
    if (posicao.y < 0)            posicao.y = ALTURA_TELA;
    if (posicao.y > ALTURA_TELA)  posicao.y = 0;
}

float RaioDoAsteroid(TamanhoAsteroid tamanho) {
    if (tamanho == GRANDE)  return 40.0f;
    if (tamanho == MEDIO)   return 22.0f;
    return 12.0f;
}

Asteroid CriarAsteroid(TamanhoAsteroid tamanho) {
    Asteroid a;
    a.tamanho = tamanho;
    a.ativo   = true;
    a.angulo  = 0.0f;
    a.velocidadeRotacao = (float)(rand() % 60 + 30) * (rand() % 2 == 0 ? 1 : -1);

    int borda = rand() % 4;
    if (borda == 0)      a.posicao = { (float)(rand() % LARGURA_TELA), 0 };
    else if (borda == 1) a.posicao = { (float)(rand() % LARGURA_TELA), (float)ALTURA_TELA };
    else if (borda == 2) a.posicao = { 0, (float)(rand() % ALTURA_TELA) };
    else                 a.posicao = { (float)LARGURA_TELA, (float)(rand() % ALTURA_TELA) };

    float direcao   = (float)(rand() % 360) * DEG2RAD;
    float velocidade = (tamanho == GRANDE) ? 60.0f : (tamanho == MEDIO ? 100.0f : 150.0f);
    a.velocidade = { cosf(direcao) * velocidade, sinf(direcao) * velocidade };

    return a;
}

void DesenharAsteroid(const Asteroid& a) {
    if (!a.ativo) return;

    float raio = RaioDoAsteroid(a.tamanho);
    int lados  = 8;
    float rad  = a.angulo * DEG2RAD;
    float irregularidade[8] = { 0.9f, 1.1f, 0.85f, 1.2f, 0.95f, 1.05f, 0.8f, 1.15f };

    for (int i = 0; i < lados; i++) {
        float ang1 = rad + (i      / (float)lados) * 2 * PI;
        float ang2 = rad + ((i+1)  / (float)lados) * 2 * PI;
        float r1   = raio * irregularidade[i % 8];
        float r2   = raio * irregularidade[(i+1) % 8];

        Vector2 p1 = { a.posicao.x + cosf(ang1) * r1, a.posicao.y + sinf(ang1) * r1 };
        Vector2 p2 = { a.posicao.x + cosf(ang2) * r2, a.posicao.y + sinf(ang2) * r2 };

        Color cor = (a.tamanho == GRANDE) ? GRAY : (a.tamanho == MEDIO ? LIGHTGRAY : WHITE);
        DrawLineV(p1, p2, cor);
    }
}

void DesenharNave(const Nave& nave) {
    float rad = nave.angulo * DEG2RAD;

    Vector2 frente   = {  0.0f, -20.0f };
    Vector2 esquerda = { -12.0f, 12.0f };
    Vector2 direita  = {  12.0f, 12.0f };

    auto Transformar = [&](Vector2 ponto) -> Vector2 {
        float rx = ponto.x * cosf(rad) - ponto.y * sinf(rad);
        float ry = ponto.x * sinf(rad) + ponto.y * cosf(rad);
        return { nave.posicao.x + rx, nave.posicao.y + ry };
    };

    DrawTriangleLines(Transformar(frente), Transformar(esquerda), Transformar(direita), SKYBLUE);

    if (nave.acelerando) {
        Vector2 chamaTopo = {  0.0f, 28.0f };
        Vector2 chamaEsq  = { -6.0f, 14.0f };
        Vector2 chamaDir  = {  6.0f, 14.0f };

        Color corChama = (GetTime() * 10 - (int)(GetTime() * 10) > 0.5f) ? ORANGE : YELLOW;
        DrawTriangle(Transformar(chamaTopo), Transformar(chamaEsq), Transformar(chamaDir), corChama);
    }
}

void Atirar(Nave& nave, std::vector<Bala>& balas) {
    // Verifica cooldown
    if (nave.tempoCooldown > 0) return;

    // Conta balas ativas
    int balasAtivas = 0;
    for (const auto& b : balas)
        if (b.ativa) balasAtivas++;
    if (balasAtivas >= MAX_BALAS) return;

    float rad = nave.angulo * DEG2RAD;

    Bala novaBala;
    novaBala.posicao   = { nave.posicao.x + sinf(rad) * 22.0f,
                           nave.posicao.y - cosf(rad) * 22.0f };
    novaBala.velocidade = { sinf(rad) * VELOCIDADE_BALA,
                           -cosf(rad) * VELOCIDADE_BALA };
    novaBala.tempoVida = TEMPO_VIDA_BALA;
    novaBala.ativa     = true;

    balas.push_back(novaBala);
    nave.tempoCooldown = COOLDOWN_TIRO;
}

// ─── Main ─────────────────────────────────────────────────────
int main(void)
{
    InitWindow(LARGURA_TELA, ALTURA_TELA, "Asteroids 2");
    SetTargetFPS(60);

    Nave nave;
    nave.posicao       = { LARGURA_TELA / 2.0f, ALTURA_TELA / 2.0f };
    nave.velocidade    = { 0.0f, 0.0f };
    nave.angulo        = 0.0f;
    nave.acelerando    = false;
    nave.tempoCooldown = 0.0f;

    std::vector<Bala>     balas;
    std::vector<Asteroid> asteroids;
    for (int i = 0; i < TOTAL_ASTEROIDS; i++)
        asteroids.push_back(CriarAsteroid(GRANDE));

    while (!WindowShouldClose())
    {
        float deltaTempo = GetFrameTime();

        // ── Controles da nave ──
        if (IsKeyDown(KEY_LEFT)  || IsKeyDown(KEY_A))
            nave.angulo -= VELOCIDADE_GIRO * deltaTempo;
        if (IsKeyDown(KEY_RIGHT) || IsKeyDown(KEY_D))
            nave.angulo += VELOCIDADE_GIRO * deltaTempo;

        nave.acelerando = IsKeyDown(KEY_UP) || IsKeyDown(KEY_W);

        if (nave.acelerando) {
            float rad = nave.angulo * DEG2RAD;
            nave.velocidade.x += sinf(rad) * EMPUXO * deltaTempo;
            nave.velocidade.y -= cosf(rad) * EMPUXO * deltaTempo;

            float rapidez = sqrtf(nave.velocidade.x * nave.velocidade.x +
                                  nave.velocidade.y * nave.velocidade.y);
            if (rapidez > VELOCIDADE_MAX) {
                nave.velocidade.x = (nave.velocidade.x / rapidez) * VELOCIDADE_MAX;
                nave.velocidade.y = (nave.velocidade.y / rapidez) * VELOCIDADE_MAX;
            }
        }

        nave.velocidade.x *= ATRITO;
        nave.velocidade.y *= ATRITO;
        nave.posicao.x    += nave.velocidade.x * deltaTempo;
        nave.posicao.y    += nave.velocidade.y * deltaTempo;
        VerificarBordas(nave.posicao);

        // Cooldown do tiro
        if (nave.tempoCooldown > 0)
            nave.tempoCooldown -= deltaTempo;

        // Atirar com Espaço
        if (IsKeyDown(KEY_SPACE))
            Atirar(nave, balas);

        // ── Atualiza balas ──
        for (auto& b : balas) {
            if (!b.ativa) continue;
            b.posicao.x  += b.velocidade.x * deltaTempo;
            b.posicao.y  += b.velocidade.y * deltaTempo;
            b.tempoVida  -= deltaTempo;

            // Desativa se sair da tela ou tempo esgotar
            if (b.tempoVida <= 0 ||
                b.posicao.x < 0 || b.posicao.x > LARGURA_TELA ||
                b.posicao.y < 0 || b.posicao.y > ALTURA_TELA)
                b.ativa = false;
        }

        // ── Atualiza asteroids ──
        for (auto& a : asteroids) {
            if (!a.ativo) continue;
            a.angulo    += a.velocidadeRotacao * deltaTempo;
            a.posicao.x += a.velocidade.x * deltaTempo;
            a.posicao.y += a.velocidade.y * deltaTempo;
            VerificarBordas(a.posicao);
        }

        // ── Desenho ──
        BeginDrawing();
            ClearBackground(BLACK);

            for (const auto& a : asteroids)
                DesenharAsteroid(a);

            // Desenha balas
            for (const auto& b : balas)
                if (b.ativa)
                    DrawCircleV(b.posicao, 3.0f, YELLOW);

            DesenharNave(nave);

            DrawText("ASTEROIDS 2", 10, 10, 20, SKYBLUE);
            DrawText("W: Acelerar  |  A: Esquerda  |  D: Direita  |  Espaco: Atirar", 10, ALTURA_TELA - 30, 16, DARKGRAY);

        EndDrawing();
    }

    CloseWindow();
    return 0;
}
