#include "raylib.h"
#include <cmath>
#include <vector>
#include <cstdlib>
#include <algorithm>
#include <fstream>
#include <string>
#include <cstring>

const int LARGURA_TELA         = 800;
const int ALTURA_TELA          = 600;
const float VELOCIDADE_GIRO    = 200.0f;
const float EMPUXO             = 300.0f;
const float ATRITO             = 0.98f;
const float VELOCIDADE_MAX     = 400.0f;
const int   ASTEROIDS_BASE     = 4;
const int   MAX_BALAS          = 12;
const float VELOCIDADE_BALA    = 550.0f;
const float TEMPO_VIDA_BALA    = 1.2f;
const float COOLDOWN_TIRO      = 0.25f;
const float TEMPO_VIDA_POWERUP = 7.0f;
const float CHANCE_POWERUP     = 0.07f;
const float INTERVALO_INIMIGO  = 20.0f;
const float COOLDOWN_INIMIGO   = 2.0f;
const float VELOCIDADE_INIMIGO = 120.0f;
const int   TOTAL_ESTRELAS     = 150;

enum TamanhoAsteroid { GRANDE=0, MEDIO, PEQUENO };
enum TipoPowerup     { TIRO_TRIPLO=0, ESCUDO, VIDA_EXTRA, TIRO_RAPIDO, LASER };
enum TelaJogo        { MENU=0, JOGANDO, PAUSADO, GAME_OVER, INSERIR_NICK, RANKING };

struct Nave {
    Vector2 posicao, velocidade;
    float   angulo, tempoCooldown, tempoTiroTriplo, tempoLaser, tempoTiroRapido;
    bool    acelerando, ativa, escudo, tiroTriplo, laser, tiroRapido;
};

struct Bala {
    Vector2 posicao, velocidade;
    float   tempoVida;
    bool    ativa, doInimigo, ehLaser;
};

struct Asteroid {
    Vector2         posicao, velocidade;
    float           angulo, velocidadeRotacao;
    TamanhoAsteroid tamanho;
    int             hp;
    bool            ativo;
    Color           cor;
};

struct Powerup {
    Vector2     posicao;
    TipoPowerup tipo;
    float       tempoVida, angulo;
    bool        ativo;
};

struct Inimigo {
    Vector2 posicao, velocidade;
    float   angulo, cooldownTiro;
    bool    ativo;
};



struct Chefe {
    Vector2 posicao, velocidade;
    float   angulo, velocidadeRotacao, cooldownTiro;
    int     hp, hpMax;
    bool    ativo;
};

struct Particula {
    Vector2 posicao, velocidade;
    float   tempoVida, tempoVidaMax, tamanho;
    Color   cor;
    bool    ativa;
};

struct Estrela {
    Vector2 posicao;
    float   brilho, velocidade, pulsacao;
};

struct Rastro {
    Vector2 posicao;
    float   tempoVida, tempoVidaMax, tamanho;
    Color   cor;
};

struct Sons {
    Sound tiro;
    Sound explosao;
    Sound explosaoGrande;
    Sound powerup;
    Music musica;
};

int CarregarHighScore() {
    std::ifstream f("highscore.dat");
    int hs=0;
    if(f.is_open()){f>>hs;f.close();}
    return hs;
}

void SalvarHighScore(int hs) {
    std::ofstream f("highscore.dat");
    if(f.is_open()){f<<hs;f.close();}
}

struct EntradaRanking {
    char nick[20];
    int  pontuacao;
};

void BubbleSort(std::vector<EntradaRanking>& r) {
    int n=r.size();
    for(int i=0;i<n-1;i++)
        for(int j=0;j<n-i-1;j++)
            if(r[j].pontuacao<r[j+1].pontuacao)
                std::swap(r[j],r[j+1]);
}

void InsertionSort(std::vector<EntradaRanking>& r) {
    int n=r.size();
    for(int i=1;i<n;i++){
        EntradaRanking chave=r[i]; int j=i-1;
        while(j>=0&&r[j].pontuacao<chave.pontuacao){r[j+1]=r[j];j--;}
        r[j+1]=chave;
    }
}

void SelectionSort(std::vector<EntradaRanking>& r) {
    int n=r.size();
    for(int i=0;i<n-1;i++){
        int maxIdx=i;
        for(int j=i+1;j<n;j++) if(r[j].pontuacao>r[maxIdx].pontuacao) maxIdx=j;
        std::swap(r[i],r[maxIdx]);
    }
}

std::vector<EntradaRanking> CarregarRanking() {
    std::vector<EntradaRanking> r;
    std::ifstream f("ranking.dat");
    if(f.is_open()){
        EntradaRanking e;
        while(f.read((char*)&e,sizeof(EntradaRanking))) r.push_back(e);
        f.close();
    }
    BubbleSort(r);
    return r;
}

void SalvarRanking(std::vector<EntradaRanking>& r) {
    BubbleSort(r);
    if(r.size()>10)r.resize(10);
    std::ofstream f("ranking.dat");
    if(f.is_open()){
        for(const auto& e:r) f.write((const char*)&e,sizeof(EntradaRanking));
        f.close();
    }
}

void AdicionarAoRanking(const char* nick, int pontuacao, std::vector<EntradaRanking>& ranking) {
    EntradaRanking e;
    strncpy(e.nick, nick, 19); e.nick[19]='\0';
    e.pontuacao=pontuacao;
    ranking.push_back(e);
    BubbleSort(ranking);
    if(ranking.size()>10)ranking.resize(10);
    SalvarRanking(ranking);
}

void VerificarBordas(Vector2& p) {
    if(p.x<0)p.x=LARGURA_TELA; if(p.x>LARGURA_TELA)p.x=0;
    if(p.y<0)p.y=ALTURA_TELA;  if(p.y>ALTURA_TELA) p.y=0;
}
float RaioDoAsteroid(TamanhoAsteroid t){return t==GRANDE?40.f:t==MEDIO?22.f:12.f;}
float Distancia(Vector2 a,Vector2 b){float dx=a.x-b.x,dy=a.y-b.y;return sqrtf(dx*dx+dy*dy);}
Color CorAleatoriaPedra(){
    int t=rand()%4;
    if(t==0)return{130,120,110,255};if(t==1)return{100,110,120,255};
    if(t==2)return{120,100,90,255};return{140,140,130,255};
}

std::vector<Estrela> CriarEstrelas(){
    std::vector<Estrela> v;
    for(int i=0;i<TOTAL_ESTRELAS;i++){
        Estrela e;
        e.posicao={(float)(rand()%LARGURA_TELA),(float)(rand()%ALTURA_TELA)};
        e.brilho=(float)(rand()%80+20)/100.f;
        e.velocidade=(float)(rand()%3+1)*5.f;
        e.pulsacao=(float)(rand()%628)/100.f;
        v.push_back(e);
    }
    return v;
}
void AtualizarEstrelas(std::vector<Estrela>& es,Vector2 vel,float dt){
    for(auto& e:es){
        float f=e.velocidade/15.f;
        e.posicao.x-=vel.x*f*dt*0.05f; e.posicao.y-=vel.y*f*dt*0.05f;
        e.pulsacao+=dt*1.5f; VerificarBordas(e.posicao);
    }
}
void DesenharEstrelas(const std::vector<Estrela>& es){
    for(const auto& e:es){
        float p=e.brilho*(0.7f+0.3f*sinf(e.pulsacao));
        Color c={200,210,255,(unsigned char)(p*255)};
        if(e.velocidade>10.f)DrawCircle((int)e.posicao.x,(int)e.posicao.y,2,c);
        else DrawPixelV(e.posicao,c);
    }
}
void DesenharNebulosa(){
    DrawCircle(150,100,180,{30,0,60,18}); DrawCircle(650,450,200,{0,20,60,15});
    DrawCircle(400,300,250,{10,30,50,10}); DrawCircle(700,100,120,{50,0,30,12});
    DrawCircle(100,500,150,{0,40,40,12});
}

void CriarExplosao(Vector2 pos,int qtd,Color cor,std::vector<Particula>& ps){
    for(int i=0;i<qtd;i++){
        float a=(float)(rand()%360)*DEG2RAD,v=(float)(rand()%120+30);
        Particula p;
        p.posicao={pos.x,pos.y}; p.velocidade={cosf(a)*v,sinf(a)*v};
        p.tempoVida=p.tempoVidaMax=(float)(rand()%60+30)/100.f;
        p.tamanho=(float)(rand()%3+1); p.cor=cor; p.ativa=true;
        ps.push_back(p);
    }
}
void CriarRastro(const Nave& nave,std::vector<Rastro>& rs){
    float rad=nave.angulo*DEG2RAD;
    Vector2 tr={nave.posicao.x-sinf(rad)*16.f,nave.posicao.y+cosf(rad)*16.f};
    Rastro r; r.posicao=tr; r.tempoVida=r.tempoVidaMax=0.3f;
    r.tamanho=(float)(rand()%4+2);
    r.cor=(rand()%3==0)?WHITE:(rand()%2==0)?ORANGE:YELLOW;
    rs.push_back(r);
}
void AtualizarParticulas(std::vector<Particula>& ps,float dt){
    for(auto& p:ps){if(!p.ativa)continue;p.posicao.x+=p.velocidade.x*dt;p.posicao.y+=p.velocidade.y*dt;p.velocidade.x*=0.95f;p.velocidade.y*=0.95f;p.tempoVida-=dt;if(p.tempoVida<=0)p.ativa=false;}
}
void AtualizarRastros(std::vector<Rastro>& rs,float dt){
    for(auto& r:rs)r.tempoVida-=dt;
    std::vector<Rastro> at; for(const auto& r:rs)if(r.tempoVida>0)at.push_back(r); rs=at;
}
void DesenharParticulas(const std::vector<Particula>& ps){
    for(const auto& p:ps){if(!p.ativa)continue;float a=p.tempoVida/p.tempoVidaMax;Color c=p.cor;c.a=(unsigned char)(a*255);DrawCircleV(p.posicao,p.tamanho*a,c);}
}
void DesenharRastros(const std::vector<Rastro>& rs){
    for(const auto& r:rs){float a=r.tempoVida/r.tempoVidaMax;Color c=r.cor;c.a=(unsigned char)(a*180);DrawCircleV(r.posicao,r.tamanho*a,c);}
}

Asteroid CriarAsteroidNaPosicao(TamanhoAsteroid tamanho,Vector2 pos,float mv=1.f){
    Asteroid a; a.tamanho=tamanho; a.ativo=true; a.angulo=0;
    a.velocidadeRotacao=(float)(rand()%60+30)*(rand()%2==0?1:-1);
    a.posicao=pos; a.cor=CorAleatoriaPedra();
    a.hp=(tamanho==GRANDE)?1:1;
    float dir=(float)(rand()%360)*DEG2RAD;
    float vel=((tamanho==GRANDE)?60.f:(tamanho==MEDIO)?100.f:150.f)*mv;
    a.velocidade={cosf(dir)*vel,sinf(dir)*vel};
    return a;
}
Asteroid CriarAsteroid(TamanhoAsteroid tamanho,float mv=1.f){
    Vector2 pos; int b=rand()%4;
    if(b==0)pos={(float)(rand()%LARGURA_TELA),0};
    else if(b==1)pos={(float)(rand()%LARGURA_TELA),(float)ALTURA_TELA};
    else if(b==2)pos={0,(float)(rand()%ALTURA_TELA)};
    else pos={(float)LARGURA_TELA,(float)(rand()%ALTURA_TELA)};
    return CriarAsteroidNaPosicao(tamanho,pos,mv);
}
void DesenharAsteroid(const Asteroid& a){
    if(!a.ativo)return;
    float raio=RaioDoAsteroid(a.tamanho); int lados=9; float rad=a.angulo*DEG2RAD;
    float ir[9]={0.9f,1.15f,0.8f,1.2f,0.95f,1.1f,0.85f,1.05f,0.9f};
    for(int i=0;i<lados;i++){
        float a1=rad+(i/(float)lados)*2*PI,a2=rad+((i+1)/(float)lados)*2*PI;
        float r1=raio*ir[i%9],r2=raio*ir[(i+1)%9];
        Vector2 p1={a.posicao.x+cosf(a1)*r1,a.posicao.y+sinf(a1)*r1};
        Vector2 p2={a.posicao.x+cosf(a2)*r2,a.posicao.y+sinf(a2)*r2};
        DrawLineV(p1,p2,a.cor);
        Vector2 c=a.posicao;
        Vector2 p1i={c.x+(p1.x-c.x)*0.6f,c.y+(p1.y-c.y)*0.6f};
        Vector2 p2i={c.x+(p2.x-c.x)*0.6f,c.y+(p2.y-c.y)*0.6f};
        Color ce={(unsigned char)(a.cor.r/2),(unsigned char)(a.cor.g/2),(unsigned char)(a.cor.b/2),80};
        DrawLineV(p1i,p2i,ce);
    }
}

void DesenharNave(const Nave& nave){
    if(!nave.ativa)return;
    float rad=nave.angulo*DEG2RAD;
    auto T=[&](Vector2 p)->Vector2{float rx=p.x*cosf(rad)-p.y*sinf(rad),ry=p.x*sinf(rad)+p.y*cosf(rad);return{nave.posicao.x+rx,nave.posicao.y+ry};};
    DrawTriangleLines(T({0,-22}),T({-13,14}),T({13,14}),SKYBLUE);
    DrawLineV(T({0,-22}),T({0,7}),{100,200,255,120});
    DrawLineV(T({-13,14}),T({0,7}),{100,200,255,80});
    DrawLineV(T({13,14}),T({0,7}),{100,200,255,80});
    DrawCircleV(T({0,-10}),3.f,{150,230,255,200});
    if(nave.escudo){float pl=0.8f+0.2f*sinf((float)GetTime()*5.f);Color ce={100,200,255,(unsigned char)(150*pl)};DrawCircleLines((int)nave.posicao.x,(int)nave.posicao.y,28.f,ce);DrawCircleLines((int)nave.posicao.x,(int)nave.posicao.y,30.f,{100,200,255,(unsigned char)(60*pl)});}
    if(nave.laser){float pl=0.8f+0.2f*sinf((float)GetTime()*8.f);DrawCircleLines((int)nave.posicao.x,(int)nave.posicao.y,20.f,{255,50,50,(unsigned char)(120*pl)});}
}

void DesenharInimigo(const Inimigo& in){
    if(!in.ativo)return;
    float pl=0.8f+0.2f*sinf((float)GetTime()*4.f);
    DrawCircleLines((int)in.posicao.x,(int)in.posicao.y,16,{255,60,60,(unsigned char)(255*pl)});
    DrawCircleLines((int)in.posicao.x,(int)in.posicao.y,10,{255,100,100,180});
    DrawCircle((int)in.posicao.x,(int)in.posicao.y,5,{255,50,50,(unsigned char)(200*pl)});
    for(int i=0;i<4;i++){float a=(i/4.f)*2*PI+(float)GetTime();Vector2 p1={in.posicao.x+cosf(a)*10,in.posicao.y+sinf(a)*10},p2={in.posicao.x+cosf(a)*16,in.posicao.y+sinf(a)*16};DrawLineV(p1,p2,{255,80,80,180});}
}



void DesenharChefe(const Chefe& ch){
    if(!ch.ativo)return;
    float pl=0.7f+0.3f*sinf((float)GetTime()*3.f);
    float raio=65.f;
    float ir[12]={0.85f,1.15f,0.9f,1.2f,0.8f,1.1f,0.95f,1.15f,0.85f,1.05f,0.9f,1.2f};
    for(int i=0;i<12;i++){
        float a1=ch.angulo*DEG2RAD+(i/12.f)*2*PI,a2=ch.angulo*DEG2RAD+((i+1)/12.f)*2*PI;
        float r1=raio*ir[i%12],r2=raio*ir[(i+1)%12];
        Vector2 p1={ch.posicao.x+cosf(a1)*r1,ch.posicao.y+sinf(a1)*r1};
        Vector2 p2={ch.posicao.x+cosf(a2)*r2,ch.posicao.y+sinf(a2)*r2};
        Color c={200,(unsigned char)(50+(int)(100*pl)),50,(unsigned char)(220*pl)};
        DrawLineV(p1,p2,c);
    }
    DrawCircleLines((int)ch.posicao.x,(int)ch.posicao.y,(int)(raio*0.5f),{255,80,50,150});
    DrawCircle((int)ch.posicao.x,(int)ch.posicao.y,12,{255,100,50,(unsigned char)(200*pl)});
    float barW=130.f,barH=8.f;
    float hpFrac=(float)ch.hp/(float)ch.hpMax;
    DrawRectangle((int)(ch.posicao.x-barW/2),(int)(ch.posicao.y-85),(int)barW,(int)barH,{60,0,0,200});
    DrawRectangle((int)(ch.posicao.x-barW/2),(int)(ch.posicao.y-85),(int)(barW*hpFrac),(int)barH,{255,60,30,255});
    const char* hpTxt=TextFormat("CHEFE %d/%d",ch.hp,ch.hpMax);
    DrawText(hpTxt,(int)(ch.posicao.x-MeasureText(hpTxt,12)/2),(int)(ch.posicao.y-100),12,{255,150,100,220});
}

void DesenharPowerup(const Powerup& p){
    if(!p.ativo)return;
    if(p.tempoVida<2.f&&(int)(p.tempoVida*5)%2==0)return;
    Color c=(p.tipo==TIRO_TRIPLO)?RED:(p.tipo==ESCUDO)?SKYBLUE:(p.tipo==VIDA_EXTRA)?GREEN:(p.tipo==TIRO_RAPIDO)?YELLOW:PURPLE;
    float pl=0.8f+0.2f*sinf((float)GetTime()*4.f);
    DrawCircleLines((int)p.posicao.x,(int)p.posicao.y,14,{c.r,c.g,c.b,(unsigned char)(150*pl)});
    DrawCircleLines((int)p.posicao.x,(int)p.posicao.y,10,c);
    DrawCircle((int)p.posicao.x,(int)p.posicao.y,5,{c.r,c.g,c.b,(unsigned char)(200*pl)});
    const char* lb=(p.tipo==TIRO_TRIPLO)?"T":(p.tipo==ESCUDO)?"E":(p.tipo==VIDA_EXTRA)?"V":(p.tipo==TIRO_RAPIDO)?"R":"L";
    DrawText(lb,(int)p.posicao.x-4,(int)p.posicao.y-6,10,WHITE);
}

void AtirarBala(Vector2 origem,float angulo,std::vector<Bala>& balas,bool doInimigo,bool ehLaser=false){
    float rad=angulo*DEG2RAD;
    Bala b;
    b.posicao={origem.x+sinf(rad)*22.f,origem.y-cosf(rad)*22.f};
    b.velocidade={sinf(rad)*VELOCIDADE_BALA,-cosf(rad)*VELOCIDADE_BALA};
    b.tempoVida=ehLaser?2.f:TEMPO_VIDA_BALA;
    b.ativa=true; b.doInimigo=doInimigo; b.ehLaser=ehLaser;
    balas.push_back(b);
}

void Atirar(Nave& nave,std::vector<Bala>& balas,Sound& somTiro){
    if(nave.tempoCooldown>0||!nave.ativa)return;
    int n=0; for(const auto& b:balas)if(b.ativa)n++;
    if(n>=MAX_BALAS)return;
    float cooldown=nave.tiroRapido?COOLDOWN_TIRO*0.4f:COOLDOWN_TIRO;
    AtirarBala(nave.posicao,nave.angulo,balas,false,nave.laser);
    if(nave.tiroTriplo){
        AtirarBala(nave.posicao,nave.angulo-20,balas,false,nave.laser);
        AtirarBala(nave.posicao,nave.angulo+20,balas,false,nave.laser);
    }
    nave.tempoCooldown=cooldown;
    PlaySound(somTiro);
}

void ReiniciarNave(Nave& nave){
    nave.posicao={LARGURA_TELA/2.f,ALTURA_TELA/2.f};
    nave.velocidade={0,0}; nave.angulo=0;
    nave.acelerando=false; nave.tempoCooldown=0;
    nave.ativa=true; nave.escudo=false;
    nave.tiroTriplo=false; nave.tempoTiroTriplo=0;
    nave.laser=false; nave.tempoLaser=0;
    nave.tiroRapido=false; nave.tempoTiroRapido=0;
}

void IniciarFase(std::vector<Asteroid>& asteroids,Chefe& chefe,int fase,float& tempoInimigo){
    asteroids.clear();
    chefe.ativo=false;
    if(fase%5==0){
        chefe.ativo=true; chefe.posicao={LARGURA_TELA/2.f,150.f};
        chefe.velocidade={(float)(rand()%60-30),30.f};
        chefe.angulo=0; chefe.velocidadeRotacao=15.f;
        chefe.cooldownTiro=2.f;
        chefe.hpMax=10+fase*2; chefe.hp=chefe.hpMax;
    } else {
        int total=ASTEROIDS_BASE+(fase-1)*2; if(total>14)total=14;
        float mv=1.f+(fase-1)*0.15f; if(mv>2.5f)mv=2.5f;
        for(int i=0;i<total;i++)asteroids.push_back(CriarAsteroid(GRANDE,mv));
    }
    tempoInimigo=INTERVALO_INIMIGO-(fase-1)*2.f; if(tempoInimigo<8.f)tempoInimigo=8.f;
}

void ReiniciarJogo(Nave& nave,std::vector<Bala>& balas,std::vector<Asteroid>& asteroids,
                   std::vector<Powerup>& powerups,std::vector<Particula>& particulas,
                   std::vector<Rastro>& rastros,Inimigo& inimigo,Chefe& chefe,
                   int& pontuacao,int& vidas,int& fase,float& tempoInvencivel,float& tempoFlash,float& tempoInimigo){
    ReiniciarNave(nave);
    balas.clear();powerups.clear();particulas.clear();rastros.clear();
    inimigo.ativo=false;chefe.ativo=false;
    pontuacao=0;vidas=3;fase=1;
    tempoInvencivel=0;tempoFlash=0;
    IniciarFase(asteroids,chefe,fase,tempoInimigo);
}

void DesenharHUD(int pontuacao,int highScore,int vidas,int fase,const Nave& nave){
    DrawRectangle(0,0,LARGURA_TELA,36,{0,0,0,100});
    DrawText("ASTEROIDS 2",10,8,20,SKYBLUE);
    const char* tp=TextFormat("PONTOS: %d",pontuacao);
    DrawText(tp,LARGURA_TELA/2-MeasureText(tp,18)/2,9,18,WHITE);
    const char* tf=TextFormat("FASE %d",fase);
    DrawText(tf,LARGURA_TELA/2-MeasureText(tf,14)/2,30,14,{150,150,255,220});
    const char* ths=TextFormat("RECORDE: %d",highScore);
    DrawText(ths,LARGURA_TELA-MeasureText(ths,14)-10,22,14,{200,180,50,200});
    for(int i=0;i<vidas;i++){int x=LARGURA_TELA-30-i*22;DrawTriangle({(float)x,(float)8},{(float)(x-7),(float)22},{(float)(x+7),(float)22},{100,200,255,180});}
    int yHud=42;
    if(nave.tiroTriplo){DrawText(TextFormat("TIRO TRIPLO: %.0fs",nave.tempoTiroTriplo),10,yHud,14,{255,100,100,220});yHud+=16;}
    if(nave.laser){DrawText(TextFormat("LASER: %.0fs",nave.tempoLaser),10,yHud,14,{200,50,255,220});yHud+=16;}
    if(nave.tiroRapido){DrawText(TextFormat("TIRO RAPIDO: %.0fs",nave.tempoTiroRapido),10,yHud,14,{255,220,50,220});yHud+=16;}
    if(nave.escudo)DrawText("ESCUDO ATIVO",10,yHud,14,{100,200,255,220});
}



int main(void)
{
    InitWindow(LARGURA_TELA,ALTURA_TELA,"Asteroids 2");
    SetExitKey(KEY_NULL);
    SetTargetFPS(60);
    InitAudioDevice();

    Sons sons;
    sons.tiro          = LoadSound("tiro.wav");
    sons.explosao      = LoadSound("explosao.wav");
    sons.explosaoGrande= LoadSound("explosao_grande.wav");
    sons.powerup       = LoadSound("powerup.wav");
    sons.musica        = LoadMusicStream("musica.mp3");
    SetSoundVolume(sons.tiro, 0.4f);
    SetSoundVolume(sons.explosao, 0.5f);
    SetSoundVolume(sons.explosaoGrande, 0.7f);
    SetSoundVolume(sons.powerup, 0.6f);
    SetMusicVolume(sons.musica, 0.3f);
    PlayMusicStream(sons.musica);

    Nave nave; ReiniciarNave(nave);
    std::vector<Bala>      balas;
    std::vector<Asteroid>  asteroids;
    std::vector<Powerup>   powerups;
    std::vector<Particula> particulas;
    std::vector<Rastro>    rastros;
    auto estrelas=CriarEstrelas();
    for(int i=0;i<6;i++)asteroids.push_back(CriarAsteroid(GRANDE,1.f));
    Inimigo inimigo; inimigo.ativo=false;
    Chefe chefe; chefe.ativo=false;

    int   pontuacao=0,highScore=CarregarHighScore(),vidas=3,fase=1;
    float tempoInvencivel=0,tempoInimigo=INTERVALO_INIMIGO,tempoFlash=0;
    float tempoPisca=0,tempoNovaFase=0;
    bool  mostraMsgFase=false;
    TelaJogo tela=MENU;

    std::vector<EntradaRanking> ranking=CarregarRanking();
    char nickJogador[20]="";
    int  nickLen=0;
    bool novoRecorde=false;

    while(!WindowShouldClose())
    {
        UpdateMusicStream(sons.musica);
        float dt=GetFrameTime();
        tempoPisca+=dt;

        if(tela==MENU){
            for(auto& a:asteroids){a.angulo+=a.velocidadeRotacao*dt;a.posicao.x+=a.velocidade.x*dt;a.posicao.y+=a.velocidade.y*dt;VerificarBordas(a.posicao);}
            AtualizarEstrelas(estrelas,{0,0},dt);
            if(IsKeyPressed(KEY_ENTER)||IsKeyPressed(KEY_SPACE)){
                ReiniciarJogo(nave,balas,asteroids,powerups,particulas,rastros,inimigo,chefe,pontuacao,vidas,fase,tempoInvencivel,tempoFlash,tempoInimigo);
                estrelas=CriarEstrelas();tela=JOGANDO;
            }
            if(IsKeyPressed(KEY_R)){tela=RANKING;}
            if(IsKeyPressed(KEY_ESCAPE))break;
            BeginDrawing();ClearBackground({2,2,15,255});DesenharNebulosa();DesenharEstrelas(estrelas);
            for(const auto& a:asteroids)DesenharAsteroid(a);
            DrawRectangle(0,0,LARGURA_TELA,ALTURA_TELA,{0,0,0,110});
            int lw=MeasureText("ASTEROIDS 2",58);
            DrawText("ASTEROIDS 2",LARGURA_TELA/2-lw/2+2,177,58,{0,80,140,180});
            DrawText("ASTEROIDS 2",LARGURA_TELA/2-lw/2,175,58,SKYBLUE);

            if(highScore>0){const char* hs=TextFormat("RECORDE: %d",highScore);DrawText(hs,LARGURA_TELA/2-MeasureText(hs,18)/2,290,18,{200,180,50,220});}
            if((int)(tempoPisca*2)%2==0){int le=MeasureText("PRESSIONE ENTER PARA JOGAR",22);DrawText("PRESSIONE ENTER PARA JOGAR",LARGURA_TELA/2-le/2,320,22,WHITE);}
            int lr2=MeasureText("R: Ranking",20);DrawText("R: Ranking",LARGURA_TELA/2-lr2/2,360,20,{150,150,255,220});
            int lsc=MeasureText("ESC: Sair",20);DrawText("ESC: Sair",LARGURA_TELA/2-lsc/2,395,20,DARKGRAY);
            EndDrawing();continue;
        }

        if(tela==GAME_OVER){
            if(pontuacao>highScore){highScore=pontuacao;SalvarHighScore(highScore);}
            AtualizarEstrelas(estrelas,{0,0},dt);
            if(IsKeyPressed(KEY_ENTER)||IsKeyPressed(KEY_SPACE)){
                tela=INSERIR_NICK;
            }
            if(IsKeyPressed(KEY_ESCAPE)){
                asteroids.clear();for(int i=0;i<6;i++)asteroids.push_back(CriarAsteroid(GRANDE,1.f));
                estrelas=CriarEstrelas();tela=MENU;
            }
            BeginDrawing();ClearBackground({2,2,15,255});DesenharNebulosa();DesenharEstrelas(estrelas);
            DrawRectangle(0,0,LARGURA_TELA,ALTURA_TELA,{0,0,0,80});
            int lg=MeasureText("GAME OVER",58);
            DrawText("GAME OVER",LARGURA_TELA/2-lg/2+2,157,58,{140,0,0,180});
            DrawText("GAME OVER",LARGURA_TELA/2-lg/2,155,58,RED);
            const char* tp=TextFormat("PONTUACAO FINAL: %d",pontuacao);DrawText(tp,LARGURA_TELA/2-MeasureText(tp,24)/2,240,24,WHITE);
            const char* tf=TextFormat("FASE ALCANCADA: %d",fase);DrawText(tf,LARGURA_TELA/2-MeasureText(tf,20)/2,275,20,{150,150,255,220});
            if(pontuacao>=highScore&&pontuacao>0){const char* nr="NOVO RECORDE!";DrawText(nr,LARGURA_TELA/2-MeasureText(nr,22)/2,310,22,{255,220,50,255});}
            else if(highScore>0){const char* hs=TextFormat("RECORDE: %d",highScore);DrawText(hs,LARGURA_TELA/2-MeasureText(hs,18)/2,310,18,{200,180,50,200});}
            if((int)(tempoPisca*2)%2==0){int le=MeasureText("ENTER: Salvar Pontuacao",22);DrawText("ENTER: Salvar Pontuacao",LARGURA_TELA/2-le/2,370,22,WHITE);}
            int lm=MeasureText("ESC: Voltar ao Menu",20);DrawText("ESC: Voltar ao Menu",LARGURA_TELA/2-lm/2,410,20,DARKGRAY);
            EndDrawing();continue;
        }

        if(tela==INSERIR_NICK){
            AtualizarEstrelas(estrelas,{0,0},dt);

            // Captura teclado para o nick
            int tecla=GetCharPressed();
            while(tecla>0){
                if(tecla>=32&&tecla<=125&&nickLen<15){
                    nickJogador[nickLen]=(char)tecla;
                    nickLen++;
                    nickJogador[nickLen]='\0';
                }
                tecla=GetCharPressed();
            }
            if(IsKeyPressed(KEY_BACKSPACE)&&nickLen>0){
                nickLen--;
                nickJogador[nickLen]='\0';
            }
            if(IsKeyPressed(KEY_ENTER)&&nickLen>0){
                AdicionarAoRanking(nickJogador,pontuacao,ranking);
                ReiniciarJogo(nave,balas,asteroids,powerups,particulas,rastros,inimigo,chefe,pontuacao,vidas,fase,tempoInvencivel,tempoFlash,tempoInimigo);
                estrelas=CriarEstrelas();
                tela=RANKING;
            }
            if(IsKeyPressed(KEY_ESCAPE)){
                asteroids.clear();for(int i=0;i<6;i++)asteroids.push_back(CriarAsteroid(GRANDE,1.f));
                estrelas=CriarEstrelas();tela=MENU;
            }

            BeginDrawing();ClearBackground({2,2,15,255});DesenharNebulosa();DesenharEstrelas(estrelas);
            DrawRectangle(0,0,LARGURA_TELA,ALTURA_TELA,{0,0,0,80});
            int lt=MeasureText("DIGITE SEU NICK",36);
            DrawText("DIGITE SEU NICK",LARGURA_TELA/2-lt/2,160,36,SKYBLUE);
            const char* pts=TextFormat("Pontuacao: %d",pontuacao);
            DrawText(pts,LARGURA_TELA/2-MeasureText(pts,22)/2,215,22,WHITE);
            // Caixa de texto
            DrawRectangle(LARGURA_TELA/2-160,270,320,50,{20,20,40,200});
            DrawRectangleLines(LARGURA_TELA/2-160,270,320,50,SKYBLUE);
            // Nick com cursor piscando
            char nickComCursor[22];
            strncpy(nickComCursor,nickJogador,20);
            if((int)(tempoPisca*2)%2==0&&nickLen<15){strcat(nickComCursor,"|");}
            int lnick=MeasureText(nickComCursor,28);
            DrawText(nickComCursor,LARGURA_TELA/2-lnick/2,282,28,WHITE);
            int le=MeasureText("ENTER: Confirmar",18);
            DrawText("ENTER: Confirmar",LARGURA_TELA/2-le/2,345,18,{150,255,150,220});
            int lesc=MeasureText("ESC: Pular",16);
            DrawText("ESC: Pular",LARGURA_TELA/2-lesc/2,375,16,DARKGRAY);
            EndDrawing();continue;
        }

        if(tela==RANKING){
            AtualizarEstrelas(estrelas,{0,0},dt);
            if(IsKeyPressed(KEY_ENTER)||IsKeyPressed(KEY_ESCAPE)||IsKeyPressed(KEY_SPACE)){
                asteroids.clear();for(int i=0;i<6;i++)asteroids.push_back(CriarAsteroid(GRANDE,1.f));
                estrelas=CriarEstrelas();tela=MENU;
            }
            BeginDrawing();ClearBackground({2,2,15,255});DesenharNebulosa();DesenharEstrelas(estrelas);
            DrawRectangle(0,0,LARGURA_TELA,ALTURA_TELA,{0,0,0,80});
            int lr=MeasureText("RANKING",48);
            DrawText("RANKING",LARGURA_TELA/2-lr/2+2,42,48,{0,80,140,180});
            DrawText("RANKING",LARGURA_TELA/2-lr/2,40,48,SKYBLUE);
            // Cabeçalho
            DrawText("#",  LARGURA_TELA/2-220,105,18,DARKGRAY);
            DrawText("NICK",LARGURA_TELA/2-180,105,18,DARKGRAY);
            DrawText("PONTUACAO",LARGURA_TELA/2+60,105,18,DARKGRAY);
            DrawLine(LARGURA_TELA/2-230,125,LARGURA_TELA/2+230,125,{60,60,100,200});
            Color coresPodio[3]={{255,215,0,255},{192,192,192,255},{205,127,50,255}};
            for(int i=0;i<(int)ranking.size()&&i<10;i++){
                Color cor=(i<3)?coresPodio[i]:WHITE;
                int y=135+i*42;
                // Destaque para novo score
                if(strcmp(ranking[i].nick,nickJogador)==0&&ranking[i].pontuacao==pontuacao)
                    DrawRectangle(LARGURA_TELA/2-230,y-4,460,38,{50,100,50,80});
                DrawText(TextFormat("%d",i+1),LARGURA_TELA/2-220,y,22,cor);
                DrawText(ranking[i].nick,LARGURA_TELA/2-180,y,22,cor);
                DrawText(TextFormat("%d",ranking[i].pontuacao),LARGURA_TELA/2+60,y,22,cor);
            }
            if(ranking.empty()){
                int lv=MeasureText("Nenhuma pontuacao ainda!",20);
                DrawText("Nenhuma pontuacao ainda!",LARGURA_TELA/2-lv/2,250,20,DARKGRAY);
            }
            int lv=MeasureText("ENTER: Voltar ao Menu",18);
            DrawText("ENTER: Voltar ao Menu",LARGURA_TELA/2-lv/2,580,18,{100,100,100,200});
            EndDrawing();continue;
        }

        if(IsKeyPressed(KEY_P)&&tela==JOGANDO){tela=PAUSADO;PauseMusicStream(sons.musica);}
        else if(IsKeyPressed(KEY_P)&&tela==PAUSADO){tela=JOGANDO;ResumeMusicStream(sons.musica);}

        if(tela==PAUSADO){
            BeginDrawing();
                ClearBackground({2,2,15,255});
                DesenharNebulosa();DesenharEstrelas(estrelas);
                for(const auto& a:asteroids)DesenharAsteroid(a);
                DesenharChefe(chefe);
                DesenharInimigo(inimigo);
                DesenharNave(nave);
                DrawRectangle(0,0,LARGURA_TELA,ALTURA_TELA,{0,0,0,150});
                int lp=MeasureText("PAUSADO",52);
                DrawText("PAUSADO",LARGURA_TELA/2-lp/2+2,202,52,{0,80,140,180});
                DrawText("PAUSADO",LARGURA_TELA/2-lp/2,200,52,SKYBLUE);
                int lc=MeasureText("P: Continuar",22);
                DrawText("P: Continuar",LARGURA_TELA/2-lc/2,290,22,WHITE);
                int lm=MeasureText("ESC: Menu",20);
                DrawText("ESC: Menu",LARGURA_TELA/2-lm/2,330,20,DARKGRAY);
                if(IsKeyPressed(KEY_ESCAPE)){
                    asteroids.clear();for(int i=0;i<6;i++)asteroids.push_back(CriarAsteroid(GRANDE,1.f));
                    estrelas=CriarEstrelas();tela=MENU;ResumeMusicStream(sons.musica);
                }
            EndDrawing();
            continue;
        }

        if(nave.ativa){
            if(IsKeyDown(KEY_LEFT)||IsKeyDown(KEY_A))nave.angulo-=VELOCIDADE_GIRO*dt;
            if(IsKeyDown(KEY_RIGHT)||IsKeyDown(KEY_D))nave.angulo+=VELOCIDADE_GIRO*dt;
            nave.acelerando=IsKeyDown(KEY_UP)||IsKeyDown(KEY_W);
            if(nave.acelerando){
                float rad=nave.angulo*DEG2RAD;
                nave.velocidade.x+=sinf(rad)*EMPUXO*dt;nave.velocidade.y-=cosf(rad)*EMPUXO*dt;
                float sp=sqrtf(nave.velocidade.x*nave.velocidade.x+nave.velocidade.y*nave.velocidade.y);
                if(sp>VELOCIDADE_MAX){nave.velocidade.x=nave.velocidade.x/sp*VELOCIDADE_MAX;nave.velocidade.y=nave.velocidade.y/sp*VELOCIDADE_MAX;}
                CriarRastro(nave,rastros);
            }
            if(IsKeyDown(KEY_SPACE))Atirar(nave,balas,sons.tiro);
        }
        nave.velocidade.x*=ATRITO;nave.velocidade.y*=ATRITO;
        nave.posicao.x+=nave.velocidade.x*dt;nave.posicao.y+=nave.velocidade.y*dt;
        VerificarBordas(nave.posicao);
        if(nave.tempoCooldown>0)nave.tempoCooldown-=dt;
        if(tempoInvencivel>0)tempoInvencivel-=dt;
        if(tempoFlash>0)tempoFlash-=dt;
        if(nave.tempoTiroTriplo>0){nave.tempoTiroTriplo-=dt;if(nave.tempoTiroTriplo<=0)nave.tiroTriplo=false;}
        if(nave.tempoLaser>0){nave.tempoLaser-=dt;if(nave.tempoLaser<=0)nave.laser=false;}
        if(nave.tempoTiroRapido>0){nave.tempoTiroRapido-=dt;if(nave.tempoTiroRapido<=0)nave.tiroRapido=false;}
        AtualizarEstrelas(estrelas,nave.velocidade,dt);
        AtualizarParticulas(particulas,dt);
        AtualizarRastros(rastros,dt);

        tempoInimigo-=dt;
        if(tempoInimigo<=0&&!inimigo.ativo){
            inimigo.ativo=true;inimigo.posicao={0,(float)(rand()%ALTURA_TELA)};
            inimigo.velocidade={VELOCIDADE_INIMIGO*(1.f+(fase-1)*0.1f),0};
            inimigo.cooldownTiro=COOLDOWN_INIMIGO;
            tempoInimigo=INTERVALO_INIMIGO-(fase-1)*2.f;if(tempoInimigo<8.f)tempoInimigo=8.f;
        }
        if(inimigo.ativo){
            inimigo.posicao.x+=inimigo.velocidade.x*dt;inimigo.posicao.y+=inimigo.velocidade.y*dt;
            float dx=nave.posicao.x-inimigo.posicao.x,dy=nave.posicao.y-inimigo.posicao.y;
            inimigo.angulo=atan2f(dx,-dy)*RAD2DEG;
            inimigo.cooldownTiro-=dt;
            if(inimigo.cooldownTiro<=0){AtirarBala(inimigo.posicao,inimigo.angulo,balas,true);inimigo.cooldownTiro=COOLDOWN_INIMIGO;}
            VerificarBordas(inimigo.posicao);
            if(inimigo.posicao.x>LARGURA_TELA+50)inimigo.ativo=false;
        }



        if(chefe.ativo){
            chefe.angulo+=chefe.velocidadeRotacao*dt;
            chefe.posicao.x+=chefe.velocidade.x*dt;chefe.posicao.y+=chefe.velocidade.y*dt;
            if(chefe.posicao.x<80||chefe.posicao.x>LARGURA_TELA-80)chefe.velocidade.x*=-1;
            if(chefe.posicao.y<80||chefe.posicao.y>ALTURA_TELA-80)chefe.velocidade.y*=-1;
            chefe.cooldownTiro-=dt;
            if(chefe.cooldownTiro<=0){
                for(int i=0;i<4;i++){float a=(i/4.f)*360.f;AtirarBala(chefe.posicao,a,balas,true);}
                chefe.cooldownTiro=2.5f;
            }
            if(chefe.hp<=0){
                chefe.ativo=false;pontuacao+=1000*fase;
                for(int i=0;i<5;i++){CriarExplosao(chefe.posicao,30,RED,particulas);CriarExplosao(chefe.posicao,20,ORANGE,particulas);}
            }
        }

        for(auto& b:balas){
            if(!b.ativa)continue;
            b.posicao.x+=b.velocidade.x*dt;b.posicao.y+=b.velocidade.y*dt;b.tempoVida-=dt;
            if(b.tempoVida<=0||b.posicao.x<0||b.posicao.x>LARGURA_TELA||b.posicao.y<0||b.posicao.y>ALTURA_TELA)b.ativa=false;
        }
        for(auto& a:asteroids){if(!a.ativo)continue;a.angulo+=a.velocidadeRotacao*dt;a.posicao.x+=a.velocidade.x*dt;a.posicao.y+=a.velocidade.y*dt;VerificarBordas(a.posicao);}
        for(auto& p:powerups){if(!p.ativo)continue;p.tempoVida-=dt;p.angulo+=90.f*dt;if(p.tempoVida<=0)p.ativo=false;}

        float mv=1.f+(fase-1)*0.15f;if(mv>2.5f)mv=2.5f;
        std::vector<Asteroid> novos;
        for(auto& b:balas){
            if(!b.ativa||b.doInimigo)continue;
            for(auto& a:asteroids){
                if(!a.ativo)continue;
                if(Distancia(b.posicao,a.posicao)<RaioDoAsteroid(a.tamanho)){
                    if(!b.ehLaser)b.ativa=false;
                    a.ativo=false;
                    int qtd=(a.tamanho==GRANDE)?25:(a.tamanho==MEDIO)?15:8;
                    CriarExplosao(a.posicao,qtd,a.cor,particulas);CriarExplosao(a.posicao,qtd/2,ORANGE,particulas);CriarExplosao(a.posicao,qtd/4,WHITE,particulas);
                    PlaySound(sons.explosao);
                    if(a.tamanho==GRANDE)pontuacao+=20*fase;
                    else if(a.tamanho==MEDIO)pontuacao+=50*fase;
                    else pontuacao+=100*fase;
                    if(a.tamanho==GRANDE){novos.push_back(CriarAsteroidNaPosicao(MEDIO,a.posicao,mv));novos.push_back(CriarAsteroidNaPosicao(MEDIO,a.posicao,mv));}
                    else if(a.tamanho==MEDIO){novos.push_back(CriarAsteroidNaPosicao(PEQUENO,a.posicao,mv));novos.push_back(CriarAsteroidNaPosicao(PEQUENO,a.posicao,mv));}
                    if((float)rand()/RAND_MAX<CHANCE_POWERUP){
                        Powerup p;p.posicao=a.posicao;p.tipo=(TipoPowerup)(rand()%5);p.tempoVida=TEMPO_VIDA_POWERUP;p.angulo=0;p.ativo=true;powerups.push_back(p);
                    }
                    if(b.ehLaser)break;
                }
            }
        }
        for(auto& n:novos)asteroids.push_back(n);

        for(auto& b:balas){
            if(!b.ativa||b.doInimigo)continue;
            if(chefe.ativo&&Distancia(b.posicao,chefe.posicao)<65.f){
                if(!b.ehLaser)b.ativa=false;
                chefe.hp--;
                CriarExplosao(b.posicao,15,RED,particulas);CriarExplosao(b.posicao,8,ORANGE,particulas);
                if(rand()%3==0){float a=(float)(rand()%360)*DEG2RAD;Vector2 sp={chefe.posicao.x+cosf(a)*50,chefe.posicao.y+sinf(a)*50};novos.push_back(CriarAsteroidNaPosicao(MEDIO,sp,mv));}
            }
        }
        for(auto& n:novos)if(!n.ativo==false)asteroids.push_back(n);



        if(nave.ativa&&tempoInvencivel<=0){
            for(auto& b:balas){
                if(!b.ativa||!b.doInimigo)continue;
                if(Distancia(b.posicao,nave.posicao)<16.f){
                    b.ativa=false;CriarExplosao(nave.posicao,20,SKYBLUE,particulas);
                    if(nave.escudo){nave.escudo=false;tempoInvencivel=1.5f;}
                    else{vidas--;tempoFlash=0.3f;PlaySound(sons.explosaoGrande);ReiniciarNave(nave);tempoInvencivel=3.f;}
                    break;
                }
            }
        }
        if(nave.ativa&&tempoInvencivel<=0){
            for(const auto& a:asteroids){
                if(!a.ativo)continue;
                if(Distancia(nave.posicao,a.posicao)<RaioDoAsteroid(a.tamanho)+14.f){
                    CriarExplosao(nave.posicao,20,SKYBLUE,particulas);
                    if(nave.escudo){nave.escudo=false;tempoInvencivel=1.5f;}
                    else{vidas--;tempoFlash=0.3f;PlaySound(sons.explosaoGrande);ReiniciarNave(nave);tempoInvencivel=3.f;}
                    break;
                }
            }
            if(chefe.ativo&&Distancia(nave.posicao,chefe.posicao)<65.f){
                CriarExplosao(nave.posicao,20,SKYBLUE,particulas);
                if(nave.escudo){nave.escudo=false;tempoInvencivel=1.5f;}
                else{vidas--;tempoFlash=0.3f;PlaySound(sons.explosaoGrande);ReiniciarNave(nave);tempoInvencivel=3.f;}
            }
        }
        if(nave.ativa&&inimigo.ativo&&tempoInvencivel<=0&&Distancia(nave.posicao,inimigo.posicao)<30.f){
            CriarExplosao(inimigo.posicao,25,RED,particulas);inimigo.ativo=false;
            if(nave.escudo){nave.escudo=false;tempoInvencivel=1.5f;}
            else{vidas--;tempoFlash=0.3f;PlaySound(sons.explosaoGrande);ReiniciarNave(nave);tempoInvencivel=3.f;}
        }

        if(inimigo.ativo){for(auto& b:balas){if(!b.ativa||b.doInimigo)continue;if(Distancia(b.posicao,inimigo.posicao)<18.f){if(!b.ehLaser)b.ativa=false;inimigo.ativo=false;pontuacao+=200*fase;CriarExplosao(inimigo.posicao,30,RED,particulas);CriarExplosao(inimigo.posicao,15,ORANGE,particulas);CriarExplosao(inimigo.posicao,10,WHITE,particulas);PlaySound(sons.explosaoGrande);break;}}}

        for(auto& p:powerups){
            if(!p.ativo)continue;
            if(Distancia(nave.posicao,p.posicao)<20.f){
                p.ativo=false;
                if(p.tipo==TIRO_TRIPLO){nave.tiroTriplo=true;nave.tempoTiroTriplo=10.f;}
                else if(p.tipo==ESCUDO)nave.escudo=true;
                else if(p.tipo==VIDA_EXTRA)vidas++;
                else if(p.tipo==TIRO_RAPIDO){nave.tiroRapido=true;nave.tempoTiroRapido=8.f;}
                else if(p.tipo==LASER){nave.laser=true;nave.tempoLaser=8.f;}
                PlaySound(sons.powerup);
            }
        }

        if(vidas<=0){
            if(pontuacao>highScore){highScore=pontuacao;SalvarHighScore(highScore);}
            nickJogador[0]='\0'; nickLen=0;
            novoRecorde=(ranking.size()<10||pontuacao>ranking.back().pontuacao||ranking.empty());
            tela=GAME_OVER;
        }

        bool algumAtivo=false;
        for(const auto& a:asteroids)if(a.ativo){algumAtivo=true;break;}
        if(!algumAtivo&&!chefe.ativo){
            fase++;IniciarFase(asteroids,chefe,fase,tempoInimigo);
            mostraMsgFase=true;tempoNovaFase=3.f;
        }

        BeginDrawing();
            ClearBackground({2,2,15,255});
            DesenharNebulosa();DesenharEstrelas(estrelas);
            if(tempoFlash>0)DrawRectangle(0,0,LARGURA_TELA,ALTURA_TELA,{255,0,0,55});
            DesenharRastros(rastros);DesenharParticulas(particulas);
            for(const auto& a:asteroids)DesenharAsteroid(a);
            DesenharChefe(chefe);
            for(const auto& p:powerups)DesenharPowerup(p);
            for(const auto& b:balas){
                if(!b.ativa)continue;
                if(b.ehLaser){DrawLineEx(b.posicao,{b.posicao.x+b.velocidade.x*0.05f,b.posicao.y+b.velocidade.y*0.05f},4,{200,50,255,255});DrawLineEx(b.posicao,{b.posicao.x+b.velocidade.x*0.05f,b.posicao.y+b.velocidade.y*0.05f},8,{200,50,255,80});}
                else if(b.doInimigo){DrawCircleV(b.posicao,3.f,{255,80,80,255});DrawCircleV(b.posicao,5.f,{255,80,80,80});}
                else{DrawCircleV(b.posicao,3.f,YELLOW);DrawCircleV(b.posicao,5.f,{255,255,100,80});}
            }
            DesenharInimigo(inimigo);

            if(!(tempoInvencivel>0&&(int)(tempoInvencivel*10)%2==0))DesenharNave(nave);
            DesenharHUD(pontuacao,highScore,vidas,fase,nave);
            if(mostraMsgFase){
                const char* mf=(fase%5==0)?"CHEFE!":TextFormat("FASE %d",fase);
                Color corFase=(fase%5==0)?RED:SKYBLUE;
                int lf=MeasureText(mf,48);float alpha=(tempoNovaFase>1.f)?1.f:tempoNovaFase;
                DrawText(mf,LARGURA_TELA/2-lf/2+2,ALTURA_TELA/2-22,48,{0,80,200,(unsigned char)(180*alpha)});
                DrawText(mf,LARGURA_TELA/2-lf/2,ALTURA_TELA/2-24,48,{corFase.r,corFase.g,corFase.b,(unsigned char)(255*alpha)});
            }
            DrawText("W: Acelerar  |  A/D: Girar  |  Espaco: Atirar  |  P: Pausar",10,ALTURA_TELA-24,14,{80,80,80,180});
        EndDrawing();
    }
    UnloadSound(sons.tiro);
    UnloadSound(sons.explosao);
    UnloadSound(sons.explosaoGrande);
    UnloadSound(sons.powerup);
    UnloadMusicStream(sons.musica);
    CloseAudioDevice();
    CloseWindow();
    return 0;
}
