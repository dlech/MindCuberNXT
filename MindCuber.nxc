/*
 * Title               : MindCuber
 *
 * Author              : David Gilday
 *
 * Copyright           : (C) 2011-2013 David Gilday
 *
 * Website             : http://mindcuber.com
 *
 * Release Information : v2.2
 *
 * Modified            : $Date: 2013-07-13 19:04:59 +0100 (Sat, 13 Jul 2013) $
 *
 * Revision            : $Revision: 6401 $
 *
 * Usage:
 *
 *   This software may be used for any non-commercial purpose providing
 *   that the original author is acknowledged.
 *
 * Disclaimer:
 *
 *   This software is provided 'as is' without warranty of any kind, either
 *   express or implied, including, but not limited to, the implied warranties
 *   of fitness for a purpose, or the warranty of non-infringement.
 *
 *-----------------------------------------------------------------------------
 * Purpose : NXT software for MindCuber
 *-----------------------------------------------------------------------------
 */

// Define NXT_8527 to run on the "Orange Box" variant of MindCuber rather
// than the default variant for NXT 2.0

//#define NXT_8527

// Define SOLVE_PATTERN and one or more PATTERN_...s to "solve" to a pattern
// If ONLY_PATTERNS is also defined, MindCuber will always create one of the
// patterns selected at random, otherwise MindCuber will solve the cube
// normally most of the time and occasionally create one of the patterns

//#define SOLVE_PATTERN
//#define ONLY_PATTERNS

#define PATTERN_CHECKERBOARD
#define PATTERN_SPOT
#define PATTERN_SNAKE
#define PATTERN_CUBES
#define PATTERN_SUPERFLIP

// Define BLACK_FACE if the cube has black stickers instead of white
// Define WHITE_LOGO if the logo on the black face is mainly white

//#define BLACK_FACE
//#define WHITE_LOGO

// Define HITECHNIC_V2 to include code to support the HiTechnic v2 color
// sensor (as an alternative the default LEGO color sensor)
// Note: this code is untested and may need modifications to make it work

//#define HITECHNIC_V2

// Define LOG to enable logging to a file to help diagnose color
// discrimination issues

//#define LOG

// Include lookup tables of moves
#include "mcmoves.h"

void banner() {
  ClearScreen();
#ifdef SOLVE_PATTERN
  TextOut(0, LCD_LINE1, "MindCuber v2.2p");
  TextOut(0, LCD_LINE2, "---------------");
#else
  TextOut(0, LCD_LINE1, "MindCuber v2.2");
  TextOut(0, LCD_LINE2, "--------------");
#endif
}

#define LCD_LINE_OFF (LCD_LINE2-LCD_LINE1)

#ifdef LOG

string       logname = "mc.log";
byte         logh;
bool         logf = true;

void log(long n) {
  if (!logf)
    Write(logh, ',');
  if (n < 0) {
    Write(logh, '-');
    n = -n;
  }
  long f = 10;
  while (n >= f)
    f *= 10;
  while (f > 1) {
    f /= 10;
    byte b = (n/f)%10+'0';
    Write(logh, b);
  }
  logf = false;
}

void logeol() {
  byte eol = 10;
  Write(logh, eol);
  logf = true;
}

#define create_log()  DeleteFile(logname); CreateFile(logname, 18000, logh)
#define close_log()   CloseFile(logh)

#else

#define log(NUM)
#define logeol()
#define create_log()
#define close_log()

#endif

#ifdef DEBUG

// Only include checks if DEBUG defined to save memory

void error(string mes, int num) {
  ClearLine(LCD_LINE1);
  TextOut(0, LCD_LINE1, "ERROR:");
  NumOut(8*6, LCD_LINE1, num);
  ClearLine(LCD_LINE2);
  TextOut(0, LCD_LINE2, mes);
  while (true)
    Wait(1);
}

#else

#define error(MES, NUM)

#endif

//-----------------------------------------------------------------------------
// Background task and routines to use color sensor as flashing light
//-----------------------------------------------------------------------------

#define S_CLR     IN_3

#define CLR_R     0
#define CLR_B     1
#define CLR_O     2
#define CLR_G     3
#define CLR_W     4
#define CLR_Y     5

#define L_DEL    1000

mutex light_mtx;
bool  light_flash = false;
int   light_clr   = CLR_R;

void color_sensor_on() {
#ifdef HITECHNIC_V2
  SetSensorLowspeed(S_CLR, true);
#else
  SetSensorColorFull(S_CLR);
  ResetSensor(S_CLR);
#endif
}

void color_sensor_off() {
#ifdef HITECHNIC_V2
  SetSensorLowspeed(S_CLR, false);
#else
  SetSensorType(S_CLR, SENSOR_TYPE_NONE);
#endif
}

task light_task() {
  long ms = CurrentTick();
  Acquire(light_mtx);
  while (light_flash) {
#ifdef HITECHNIC_V2
#else
    switch (light_clr) {
    case CLR_R: SetSensorColorRed(S_CLR);   break;
    case CLR_G: SetSensorColorGreen(S_CLR); break;
    case CLR_B: SetSensorColorBlue(S_CLR);  break;
    }
#endif
    color_sensor_off();
    Release(light_mtx);
    ms += L_DEL;
    long del = ms-CurrentTick();
    if (del < 1)
      del = 1;
    else if (del > L_DEL)
      del = L_DEL;
    Wait(del);
    Acquire(light_mtx);
  }
  Release(light_mtx);
}

void flash_on(int clr) {
  Acquire(light_mtx);
  light_clr = clr;
  if (!light_flash) {
    light_flash = true;
    start light_task;
  }
  Release(light_mtx);
}

void flash_red() {
  flash_on(CLR_R);
}

void flash_green() {
  flash_on(CLR_G);
}

void flash_blue() {
  flash_on(CLR_B);
}

void flash_off() {
  Acquire(light_mtx);
  light_flash = false;
  Release(light_mtx);
  color_sensor_off();
}

//-----------------------------------------------------------------------------
// Background task and routines to display timer during solve
//-----------------------------------------------------------------------------

mutex  tmr_mtx;
bool   tmr_on   = false;
long   tmr_time = 0;
string tmr_str  = "      ";

string ms_to_str(long ms) {
  long t = ms / 100;
  long s = t / 10;
  t %= 10;
  long m = s / 60;
  s %= 60;
  return NumToStr(m)+":"+NumToStr(s/10)+NumToStr(s%10)+"."+NumToStr(t);
}

task timer_task() {
  bool on = false;
  do {
    long del = 100;
    Acquire(tmr_mtx);
    on = tmr_on;
    if (on) {
      long t = CurrentTick()-tmr_time;
      del -= (t % 100);
      tmr_str = ms_to_str(t);
      TextOut(0, LCD_LINE8, tmr_str);
    }
    Release(tmr_mtx);
    Wait(del);
  } while (on);
}

void timer_start() {
  Acquire(tmr_mtx);
  if (!tmr_on) {
    tmr_on   = true;
    tmr_time = CurrentTick();
    start timer_task;
  }
  Release(tmr_mtx);
}

void timer_stop() {
  Acquire(tmr_mtx);
  tmr_on = false;
  Release(tmr_mtx);
}

//-----------------------------------------------------------------------------
// Routines to detect the presence of a cube using the ultrasonic sensor
//-----------------------------------------------------------------------------

#define S_ULTRA   IN_2

#define CD_DIST   18
#define CD_DELAY  200
#define CD_COUNT  5

bool cd_on    = false;
long cd_time  = 0;
int  cd_count = 0;

bool check_cube_detect() {
  if (!cd_on) {
    cd_on    = true;
    cd_time  = CurrentTick();
    cd_count = 0;
    SetSensorUltrasonic(S_ULTRA);
    ResetSensor(S_ULTRA);
  }
  long diff = CurrentTick()-cd_time;
  if (diff > 0) {
    cd_time += CD_DELAY;
    return true;
  }
  return false;
}

void cube_detect_off() {
  cd_on = false;
  SetSensorType(S_ULTRA, SENSOR_TYPE_NONE);
}

bool cube_present() {
  if (check_cube_detect()) {
    if (SensorUS(S_ULTRA) < CD_DIST)
      cd_count ++;
    else
      cd_count = 0;
  }
  return cd_count >= CD_COUNT;
}

bool cube_absent() {
  if (check_cube_detect()) {
    if (SensorUS(S_ULTRA) >= CD_DIST)
      cd_count ++;
    else
      cd_count = 0;
  }
  return cd_count >= CD_COUNT;
}

//-----------------------------------------------------------------------------
// Routines to detect button presses with auto-repeat when held
//-----------------------------------------------------------------------------

#define BTN_AUTO_REPEAT_FIRST   1000
#define BTN_AUTO_REPEAT         30

long btn_time   = 0;
bool btn_left   = false;
bool btn_right  = false;
bool btn_center = false;

bool btn_auto_repeat(byte btn, bool &btn_pressed) {
  if (ButtonPressed(btn, false)) {
    if (btn_pressed) {
      long diff = CurrentTick()-btn_time;
      if (diff >= 0) {
        btn_time += BTN_AUTO_REPEAT;
        return true;
      }
    } else {
      btn_pressed = true;
      btn_time = CurrentTick()+BTN_AUTO_REPEAT_FIRST;
      return true;
    }
  } else
    btn_pressed = false;
  return false;
}

bool left_pressed() {
  return btn_auto_repeat(BTNLEFT, btn_left);
}

bool right_pressed() {
  return btn_auto_repeat(BTNRIGHT, btn_right);
}

bool center_pressed() {
  return btn_auto_repeat(BTNCENTER, btn_center);
}

//-----------------------------------------------------------------------------
// Background task and low level routines to control motors
//-----------------------------------------------------------------------------

#define NMOTORS 3

#define M_DELAY 10
#define M_SCALE 12
#define AMAX    24
#define VMAX    100

#define P_LOW   35
#define P_HIGH  87

mutex      motor_mtx;
const byte mmot[] = {OUT_A, OUT_B, OUT_C};
bool       mon[]  = {false, false, false};
bool       mgo[]  = {false, false, false};
bool       mup[]  = {false, false, false};
long       mtx[]  = {0, 0, 0};
long       mx[]   = {0, 0, 0};
long       mv[]   = {0, 0, 0};
long       ma[]   = {0, 0, 0};
long       mp[]   = {0, 0, 0};
long       me[]   = {0, 0, 0};

void motors_on() {
  Acquire(motor_mtx);
  for (int m = 0; m < NMOTORS; m++)
    mon[m] = true;
  Release(motor_mtx);
}

void motors_off() {
  bool on;
  do {
    on = false;
    Acquire(motor_mtx);
    for (int m = 0; m < NMOTORS; m++) {
      if (mon[m]) {
        if (mgo[m]) {
          on = true;
        } else {
          byte mot = mmot[m];
          OffEx(mot, RESET_NONE);
          mon[m] = false;
          mv[m]  = 0;
          ma[m]  = 0;
          mp[m]  = 0;
        }
      }
    }
    Release(motor_mtx);
    Wait(1);
  } while (on);
}

void endstop(int m) {
  long p0, p1;
  byte mot = mmot[m];
  Acquire(motor_mtx);
  mon[m] = false;
  Release(motor_mtx);
  OnFwdEx(mot, P_LOW, RESET_NONE);
  p1 = MotorTachoCount(mot);
  do {
    Wait(500);
    p0 = p1;
    p1 = MotorTachoCount(mot);
  } while (p0 < p1);
  ResetTachoCount(mot);
  Wait(200);
}

task motor_task() {
  long ms = CurrentTick();
  while (true) {
    Acquire(motor_mtx);
    for (int m = 0; m < NMOTORS; m++) {
      if (mon[m]) {
        bool rev = false;
        byte mot = mmot[m];
        long x = mx[m];
        long v = mv[m];
        long a = ma[m];
        long p = mp[m];
        long e = 0;
        long ax = M_SCALE*MotorTachoCount(mot);
        long ex = ax-x;
        if (-M_SCALE < ex && ex < M_SCALE)
          ax = x;
        else if (mgo[m])
          e  = me[m]-ex;
        long d = mtx[m]-ax;
        if (mup[m] ? (d < M_SCALE) : (d > -M_SCALE)) {
          mgo[m] = false;
          e = 0;
        }
        if (d < 0) {
          d = -d;
          v = -v;
          a = -a;
          p = -p;
          e = -e;
          rev = true;
        }
        if (d >= M_SCALE) {
          if (v >= 0) {
            long dd = (v+AMAX/2)+(v*v)/(2*AMAX);
            if (d >= dd) {
              p = P_HIGH;
              a = (AMAX*(VMAX-v))/VMAX;
              e = 0;
            } else {
              a = -(v*v)/(2*d);
              if (a < -v)
                a = -v;
              if (a < -AMAX)
                a = -AMAX;
              p = (P_HIGH*a*(VMAX-v))/(AMAX*VMAX);
            }
          } else {
            a = -v;
            if (a > AMAX)
              a = AMAX;
            p = (P_HIGH*a*(VMAX+v))/(AMAX*VMAX);
          }
        } else {
          a = -v;
          if (a < -AMAX)
            a = -AMAX;
          else if (a > AMAX)
            a = AMAX;
          p = (P_HIGH*a)/AMAX;
        }
        d = v+a/2;
        v += a;
        p += e;
        if (p > P_HIGH) {
          p = P_HIGH;
          e = 0;
        } else if (p < -P_HIGH) {
          p = -P_HIGH;
          e = 0;
        }
        if (rev) {
          d = -d;
          v = -v;
          a = -a;
          p = -p;
          e = -e;
        }
        if (p != mp[m]) {
          if (p > 0)
            OnFwdEx(mot, p, RESET_NONE);
          else if (p < 0)
            OnRevEx(mot, -p, RESET_NONE);
          else
            OffEx(mot, RESET_NONE);
          mp[m] = p;
        }
        mx[m] = ax+d;
        mv[m] = v;
        ma[m] = a;
        me[m] = e;
      }
    }
    Release(motor_mtx);
    ms += M_DELAY;
    long del = ms-CurrentTick();
    if (del < 1)
      del = 1;
    else if (del > M_DELAY)
      del = M_DELAY;
    Wait(del);
  }
}

void move_wait(int m) {
  bool go;
  do {
    Wait(1);
    Acquire(motor_mtx);
    go = mgo[m];
    Release(motor_mtx);
  } while (go);
}

void move_abs(int m, long pos) {
  Acquire(motor_mtx);
  mtx[m] = -M_SCALE*pos;
  mup[m] = (mtx[m] > mx[m]);
  mon[m] = true;
  mgo[m] = true;
  Release(motor_mtx);
}

void move_rel(int m, long pos, long off = 0) {
  Acquire(motor_mtx);
  mtx[m] += -M_SCALE*(pos+off);
  mup[m] = (mtx[m] > mx[m]);
  mon[m] = true;
  mgo[m] = true;
  Release(motor_mtx);
  if (off != 0) {
    move_wait(m);
    Acquire(motor_mtx);
    mtx[m] -= -M_SCALE*off;
    mup[m] = (mtx[m] > mx[m]);
    mgo[m] = true;
    Release(motor_mtx);
  }
}

//-----------------------------------------------------------------------------
// Routines for high level motor control for turn, scan and tilt movements
//-----------------------------------------------------------------------------

#define M_TURN  0
#define M_SCAN  1
#define M_TILT  2

#ifdef NXT_8527
#define T_90    210
#define T_CUT   7
#define T_OVR   48
#ifdef HITECHNIC_V2
#define T_SOFF  0
#define T_SCNT  15
#define T_SCNR  100
#define T_SEDG  83
#else
#define T_SOFF  12
#define T_SCNT  15
#define T_SCNR  105
#define T_SEDG  88
#endif
#define T_TILT  85
#define T_TAWY  25
#define T_TREL  62
#define T_THLD  160
#define T_ADJ   2
#else
#define T_90    270
#define T_CUT   10
#define T_OVR   57
#ifdef HITECHNIC_V2
#define T_SOFF  0
#define T_SCNT  10
#define T_SCNR  103
#define T_SEDG  87
#else
#define T_SOFF  10
#define T_SCNT  10
#define T_SCNR  103
#define T_SEDG  87
#endif
#define T_TILT  85
#define T_TAWY  25
#define T_TREL  62
#define T_THLD  160
#define T_ADJ   3
#endif

int apply_sign(int n, int d) {
  return (n < 0) ? -d : ((n > 0) ? d : 0);
}

void turn(int r) {
  move_rel(M_TURN, r*T_90);
  move_wait(M_TURN);
}

void turn45(void) {
  move_rel(M_TURN, -T_90/2);
}

void turn_face(int r, int rn) {
  move_rel(M_TURN, r*T_90, apply_sign(r, T_OVR)+apply_sign(rn, T_CUT));
  move_wait(M_TURN);
}

void scan_to_center() {
  move_abs(M_SCAN, T_SCNT);
  move_wait(M_SCAN);
}

void scan_to_corner() {
  move_abs(M_SCAN, T_SCNR);
}

void scan_to_edge() {
  move_abs(M_SCAN, T_SEDG);
}

void scan_away() {
  move_abs(M_SCAN, 200);
}

void scan_wait() {
  move_wait(M_SCAN);
}

void scan_turn_wait() {
  move_wait(M_SCAN);
  move_wait(M_TURN);
}

bool holding = false;

void tilt_away() {
  holding = false;
  move_abs(M_TILT, T_TAWY);
  move_wait(M_TILT);
}

void tilt_release() {
  holding = false;
  move_abs(M_TILT, T_TREL);
  move_wait(M_TILT);
}

void tilt_hold() {
  holding = true;
  move_abs(M_TILT, T_THLD);
  move_wait(M_TILT);
}

void tilt(int n = 1) {
  if (holding) {
    Wait(100);
  } else {
    tilt_hold();
    Wait(350);
  }
  for (int i = 0; i < n; i++) {
    if (i > 0) {
      move_wait(M_TILT);
      Wait(500);
    }
    int t = T_TILT;
    move_rel(M_TILT, t);
    move_wait(M_TILT);
    Wait(150);
    move_rel(M_TILT, -t, -25);
    move_wait(M_TILT);
  }
}

//-----------------------------------------------------------------------------
// Routines for color space conversion and color sorting
//-----------------------------------------------------------------------------

#define NFACE       6                // number of faces in cube
#define POS(FF, OO) (((FF)*8)+(OO))

const long cmax = 1024;

int  sc_r[NFACE*9];
int  sc_g[NFACE*9];
int  sc_b[NFACE*9];
int  sc_h[NFACE*9];
int  sc_s[NFACE*9];
int  sc_l[NFACE*9];
int  sc_sl[NFACE*9];
char sc_clr[NFACE*9];

void set_rgb(int o, int &rgb[]) {
  long r = rgb[0];
  long g = rgb[1];
  long b = rgb[2];
  // Convert to hsl
  long h = 0;
  long s = 0;
  long l = 0;
  long sl = 0;
  long v = r; if (g > v) v = g; if (b > v) v = b;
  long m = r; if (g < m) m = g; if (b < m) m = b;
  l = (v+m)/2;
  if (l > 0) {
    long vm = v-m;
    s = vm;
    if (s > 0) {
      long vf = v+m;
      if (l <= (cmax/2))
        vf = (2*cmax)-vf;
      long r2 = (v-r)*vf/vm;
      long g2 = (v-g)*vf/vm;
      long b2 = (v-b)*vf/vm;
      if (r == v)
        h = (g == m) ? 5*cmax+b2 : 1*cmax-g2;
      else if (g == v)
        h = (b == m) ? 1*cmax+r2 : 3*cmax-b2;
      else
        h = (r == m) ? 3*cmax+g2 : 5*cmax-r2;
      h += cmax; // rotate so R/B either side of 0
      h /= 6;
      if (h < 0)     h += cmax;
      if (h >= cmax) h -= cmax;
      sl = s*cmax/l;
    }
  }
  sc_r[o]  = r;
  sc_g[o]  = g;
  sc_b[o]  = b;
  sc_h[o]  = h;
  sc_s[o]  = s;
  sc_l[o]  = l;
  sc_sl[o] = sl;
  log(r);
  log(g);
  log(b);
  log(h);
  log(s);
  log(l);
  log(sl);
  logeol();
}

int clr_ratio(long c0, long c1) {
  int ratio = 0;
  if (c0 < c1)
    ratio = -(2000*(c1-c0)/(c1+c0));
  else if (c0 > c1)
    ratio =  (2000*(c0-c1)/(c1+c0));
  return ratio;
}

#define CMP_H    0
#define CMP_S    1
#define CMP_SL   2
#define CMP_SLR  3
#define CMP_L    4
#define CMP_LR   5
#define CMP_R_G  6
#define CMP_R_B  7
#define CMP_B_G  8

bool compare_clrs(const int cmp_fn, const int c0, const int c1) {
  bool cmp = false;
  switch (cmp_fn) {
  case CMP_H:   cmp = (sc_h[c1]  > sc_h[c0]);  break;
  case CMP_S:   cmp = (sc_s[c1]  > sc_s[c0]);  break;
  case CMP_SL:  cmp = (sc_sl[c1] > sc_sl[c0]); break;
  case CMP_SLR: cmp = (sc_sl[c1] < sc_sl[c0]); break;
  case CMP_L:   cmp = (sc_l[c1]  > sc_l[c0]);  break;
  case CMP_LR:  cmp = (sc_l[c1]  < sc_l[c0]);  break;
  case CMP_R_G: cmp = (clr_ratio(sc_r[c1], sc_g[c1])
                       < clr_ratio(sc_r[c0], sc_g[c0])); break;
  case CMP_R_B: cmp = (clr_ratio(sc_r[c1], sc_b[c1])
                       < clr_ratio(sc_r[c0], sc_b[c0])); break;
  case CMP_B_G: cmp = (clr_ratio(sc_b[c1], sc_g[c1])
                       < clr_ratio(sc_b[c0], sc_g[c0])); break;
  default:      error("cmp_fn", cmp_fn); break;
  }

  return cmp;
}

void sort_clrs(int &co[], const int s, const int n, const int cmp_fn) {
  const int e = s+n-2;
  int is = s;
  int ie = e;
  do {
    int il = e+2;
    int ih = s-2;
    for (int i = is; i <= ie; i++) {
      if (compare_clrs(cmp_fn, co[i+1], co[i])) {
        int o   = co[i];
        co[i]   = co[i+1];
        co[i+1] = o;
        if (i < il) il = i;
        if (i > ih) ih = i;
      }
    }
    is = il-1; if (is < s) is = s;
    ie = ih+1; if (ie > e) ie = e;
  } while (is <= ie);
}

void sort_colors(int &co[], const int t, const int s) {
#ifdef BLACK_FACE
  sort_clrs(co, 0*s, 6*s,
#ifdef WHITE_LOGO
	    (s == 1) ?
	    // Saturation
	    CMP_SL :
#endif
	    // Saturation (un-weighted by Lightness for black)
	    CMP_S);
#else
  // Lightness
  sort_clrs(co, 0*s, 6*s, CMP_LR);
  // Saturation
  sort_clrs(co, 0*s, 3*s, CMP_SL);
#endif // BLACK_FACE
  // Hue
  sort_clrs(co, 1*s, 5*s, CMP_H);
  // Red/Orange
  switch (t) {
  case 0: /* already sorted by hue */       break;
  case 1: sort_clrs(co, 1*s, 2*s, CMP_R_G); break;
  case 2: sort_clrs(co, 1*s, 2*s, CMP_B_G); break;
  case 3: sort_clrs(co, 1*s, 2*s, CMP_R_B); break;
  case 4: sort_clrs(co, 1*s, 2*s, CMP_SLR); break;
  case 5: sort_clrs(co, 1*s, 2*s, CMP_L);   break;
  }
  int i;
  for (i = 0; i < s; i++)
    sc_clr[co[i]] = CLR_W;
  for (; i < 2*s; i++)
    sc_clr[co[i]] = CLR_R;
  for (; i < 3*s; i++)
    sc_clr[co[i]] = CLR_O;
  for (; i < 4*s; i++)
    sc_clr[co[i]] = CLR_Y;
  for (; i < 5*s; i++)
    sc_clr[co[i]] = CLR_G;
  for (; i < 6*s; i++)
    sc_clr[co[i]] = CLR_B;
}

//-----------------------------------------------------------------------------
// Routines to scan a face of the cube and calibrate white balance
//-----------------------------------------------------------------------------

long         white_rgb[] = {910, 960, 1024};
unsigned int rgbn[INPUT_NO_OF_COLORS];

void sample_rgb(int &rgb[], int f, int o, bool cal) {
  const int scan_max = 5;
  Wait(10);
  for (int i = 0; i < 3; i++)
    rgb[i] = 0;
  for (int n = 0; n < scan_max; n++) {
#ifdef HITECHNIC_V2
    unsigned int r, g, b, w;
    do {
      Wait(1);
    } while (!ReadSensorHTRawColor2(S_CLR, r, g, b, w));
    rgbn[INPUT_RED]   = r / 4;
    rgbn[INPUT_GREEN] = g / 64;
    rgbn[INPUT_BLUE]  = (b > 20000) ? (b-20000)/48 : 0;
    rgbn[INPUT_BLANK] = w / 64;
#else
    do {
      Wait(1);
    } while (!ReadSensorColorRaw(S_CLR, rgbn));
#endif
    rgb[0] += rgbn[INPUT_RED];
    rgb[1] += rgbn[INPUT_GREEN];
    rgb[2] += rgbn[INPUT_BLUE];
    log(rgbn[INPUT_RED]);
    log(rgbn[INPUT_GREEN]);
    log(rgbn[INPUT_BLUE]);
    log(rgbn[INPUT_BLANK]);
    logeol();
  }
  for (int i = 0; i < 3; i++) {
    if (cal) {
      rgb[i] /= scan_max;
      white_rgb[i] += rgb[i];
    } else {
      rgb[i] = (white_rgb[i]*rgb[i])/(scan_max*cmax);
    }
  }
}

int scan_rgb[3];

void scan_face(int n, int f, int o, bool cal = false) {
  TextOut(0, LCD_LINE6, "Face");
  NumOut(5*6, LCD_LINE6, n);
  log(f);
  logeol();
  tilt_release();
  move_rel(M_TURN, T_SOFF);
  scan_to_center();
  sample_rgb(scan_rgb, f, 8, cal);
  int orgb = f*9+8;
  for (int i = 0; i < 4; i++) {
    turn45();
    scan_to_corner();
    set_rgb(orgb, scan_rgb);
    scan_turn_wait();
    sample_rgb(scan_rgb, f, o, cal);
    orgb = f*9+o;
    turn45();
    scan_to_edge();
    set_rgb(orgb, scan_rgb);
    scan_turn_wait();
    sample_rgb(scan_rgb, f, o+1, cal);
    orgb += 1;
    o = (o+2)&7;
  }
  move_rel(M_TURN, -T_SOFF);
  if (n != 6)
    scan_away();
  set_rgb(orgb, scan_rgb);
}

string cfg_name = "mccfg.dat";
byte   cfgh;

void calibrate_white() {
  flash_off();
  color_sensor_on();
  for (int i = 0; i < 3; i++)
    white_rgb[i] = 0;
  scan_face(1, 3, 2, true);
  long cmin = white_rgb[0];
  for (int i = 1; i < 3; i++) {
    if (white_rgb[i] < cmin)
      cmin = white_rgb[i];
  }
  for (int i = 0; i < 3; i++) {
    white_rgb[i] = (cmax*cmin)/white_rgb[i];
    NumOut(5*i*6, LCD_LINE5, white_rgb[i]);
  }
  // Save white calibration value to a configuration file
  DeleteFile(cfg_name);
  if (CreateFile(cfg_name, 100, cfgh) == LDR_SUCCESS) {
    Write(cfgh, white_rgb);
    CloseFile(cfgh);
  }
  flash_blue();
}

//-----------------------------------------------------------------------------
// Face indices
//-----------------------------------------------------------------------------

#define U        0
#define F        1
#define D        2
#define B        3
#define R        4
#define L        5

//-----------------------------------------------------------------------------
// Sequences of moves for patterns
//-----------------------------------------------------------------------------

#ifdef SOLVE_PATTERN

#ifdef PATTERN_CHECKERBOARD
const byte checkerboard_fce[] = { U, D, F, B, L, R};
const char checkerboard_rot[] = { 2, 2, 2, 2, 2, 2};
#endif

#ifdef PATTERN_SPOT
const byte spot_fce[] = { L, R, F, B, U, D, L, R};
const char spot_rot[] = {-1, 1, 1,-1, 1,-1,-1, 1};
#endif

#ifdef PATTERN_SNAKE
const byte snake_fce[] = { F, L, U, L, B, U, B, L, U, L, F, L, U, F};
const char snake_rot[] = {-1,-1, 1, 1, 2, 1, 2, 1,-1, 1,-1, 2,-1, 2};
#endif

#ifdef PATTERN_CUBES
const byte cubes_fce[] = { U, R, D, F, R, U, R, U, R, U, R, L, D, L, F, D, R};
const char cubes_rot[] = {-1, 1,-1,-1, 1, 2, 2,-1,-1, 1, 2, 1,-1,-1, 2, 2,-1};
#endif

#ifdef PATTERN_SUPERFLIP
const byte superflip_fce[] = { U, F, R, U, R, D, F, B, U, F, R, L, F, D, B, R, F, D, F, D};
const char superflip_rot[] = {-1, 2,-1, 2, 2, 2, 1,-1,-1, 2, 1, 1,-1, 2,-1, 2,-1, 2,-1,-1};
#endif

#endif

//-----------------------------------------------------------------------------
// Routines to display, transform and solve a cube
//-----------------------------------------------------------------------------

// Target moves for solve to finish early to save time
// Average: 42 moves in 5 seconds
// Range:   about 37 to 47 moves in 1 to 10 seconds

#define TARGET  42

#define NPIECE   3    // maximum number of pieces indexed in one phase
#define MV_MAX   100  // maximum number of moves per solve

#define RFIX(RR) ((((RR)+1)&3)-1)  // Normalise to range -1 to 2

#define CHR_R    'r'
#define CHR_B    'b'
#define CHR_O    'o'
#define CHR_G    'g'
#define CHR_W    'w'
#define CHR_Y    'y'

char clr_chr[] = {CHR_R, CHR_B, CHR_O, CHR_G, CHR_W, CHR_Y};

void display_face(int x, int y, byte &cube[], int f, int o) {
  byte str[4];
  str[3] = 0;
  int b = f * 8;
  str[0] = clr_chr[cube[b+  o      ]];
  str[1] = clr_chr[cube[b+ (o+1)   ]];
  str[2] = clr_chr[cube[b+((o+2)&7)]];
  TextOut(x, y, ByteArrayToStr(str));
  str[0] = clr_chr[cube[b+((o+7)&7)]];
  str[1] = clr_chr[f];
  str[2] = clr_chr[cube[b+((o+3)&7)]];
  TextOut(x, y+LCD_LINE_OFF, ByteArrayToStr(str));
  str[0] = clr_chr[cube[b+((o+6)&7)]];
  str[1] = clr_chr[cube[b+((o+5)&7)]];
  str[2] = clr_chr[cube[b+((o+4)&7)]];
  TextOut(x, y+2*LCD_LINE_OFF, ByteArrayToStr(str));
}

void display_cube(byte &cube[]) {
  display_face(0,  LCD_LINE3, cube, U, 2);
  display_face(19, LCD_LINE3, cube, F, 2);
  display_face(38, LCD_LINE3, cube, D, 2);
  display_face(38, LCD_LINE6, cube, L, 4);
  display_face(57, LCD_LINE6, cube, B, 0);
  display_face(76, LCD_LINE6, cube, R, 4);
}

void init_cube(byte &cube[]) {
  int o = 0;
  for (byte f = 0; f < NFACE; f++)
    for (int i = 0; i < 8; i++)
      cube[o++] = f;
}

void copy_cube(byte &cube0[], byte &cube1[]) {
  int o = 0;
  for (byte f = 0; f < NFACE; f++)
    for (int i = 0; i < 8; i++) {
      cube0[o] = cube1[o];
      o ++;
    }
}

bool solved(byte &cube[]) {
  int o = 0;
  for (byte f = 0; f < NFACE; f++)
    for (int i = 0; i < 8; i++) {
      if (cube[o++] != f)
        return false;
    }
  return true;
}

int  mv_n;
byte mv_f[MV_MAX];
int  mv_r[MV_MAX];

const byte opposite[] = {D, B, U, F, L, R};

void add_mv(int f, int r) {
  int  i   = mv_n;
  bool mrg = false;
  while (i > 0) {
    i--;
    int fi = mv_f[i];
    if (f == fi) {
      r += mv_r[i];
      r = RFIX(r);
      if (r != 0) {
        mv_r[i] = r;
      } else {
        mv_n --;
        while (i < mv_n) {
          mv_f[i] = mv_f[i+1];
          mv_r[i] = mv_r[i+1];
          i++;
        }
      }
      mrg = true;
      break;
    }
    if (opposite[f] != fi)
      break;
  }
  if (!mrg) {
    mv_f[mv_n] = f;
    mv_r[mv_n] = RFIX(r);
    mv_n++;
  }
}

void rot_edges(byte &cube[], int r,
               int f0, int o0, int f1, int o1,
               int f2, int o2, int f3, int o3) {
  f0 *= 8;
  f1 *= 8;
  f2 *= 8;
  f3 *= 8;
  o0 += f0;
  o1 += f1;
  o2 += f2;
  o3 += f3;
  byte p;
  switch (r) {
  case 1:
    p = cube[o3];
    cube[o3] = cube[o2];
    cube[o2] = cube[o1];
    cube[o1] = cube[o0];
    cube[o0] = p;
    o0 ++; o1 ++; o2 ++; o3 ++;
    p = cube[o3];
    cube[o3] = cube[o2];
    cube[o2] = cube[o1];
    cube[o1] = cube[o0];
    cube[o0] = p;
    o0 = f0+((o0+1)&7); o1 = f1+((o1+1)&7);
    o2 = f2+((o2+1)&7); o3 = f3+((o3+1)&7);
    p = cube[o3];
    cube[o3] = cube[o2];
    cube[o2] = cube[o1];
    cube[o1] = cube[o0];
    cube[o0] = p;
    break;

  case 2:
    p = cube[o0];
    cube[o0] = cube[o2];
    cube[o2] = p;
    p = cube[o1];
    cube[o1] = cube[o3];
    cube[o3] = p;
    o0 ++; o1 ++; o2 ++; o3 ++;
    p = cube[o0];
    cube[o0] = cube[o2];
    cube[o2] = p;
    p = cube[o1];
    cube[o1] = cube[o3];
    cube[o3] = p;
    o0 = f0+((o0+1)&7); o1 = f1+((o1+1)&7);
    o2 = f2+((o2+1)&7); o3 = f3+((o3+1)&7);
    p = cube[o0];
    cube[o0] = cube[o2];
    cube[o2] = p;
    p = cube[o1];
    cube[o1] = cube[o3];
    cube[o3] = p;
    break;

  case 3:
    p = cube[o0];
    cube[o0] = cube[o1];
    cube[o1] = cube[o2];
    cube[o2] = cube[o3];
    cube[o3] = p;
    o0 ++; o1 ++; o2 ++; o3 ++;
    p = cube[o0];
    cube[o0] = cube[o1];
    cube[o1] = cube[o2];
    cube[o2] = cube[o3];
    cube[o3] = p;
    o0 = f0+((o0+1)&7); o1 = f1+((o1+1)&7);
    o2 = f2+((o2+1)&7); o3 = f3+((o3+1)&7);
    p = cube[o0];
    cube[o0] = cube[o1];
    cube[o1] = cube[o2];
    cube[o2] = cube[o3];
    cube[o3] = p;
    break;

  default:
    error("rot_edges", r);
  }
}

void rotate(byte &cube[], int f, int r) {
  r &= 3;
  switch (f) {
  case U:  rot_edges(cube, r, B, 4, R, 0, F, 0, L, 0); break;
  case F:  rot_edges(cube, r, U, 4, R, 6, D, 0, L, 2); break;
  case D:  rot_edges(cube, r, F, 4, R, 4, B, 0, L, 4); break;
  case B:  rot_edges(cube, r, D, 4, R, 2, U, 0, L, 6); break;
  case R:  rot_edges(cube, r, U, 2, B, 2, D, 2, F, 2); break;
  case L:  rot_edges(cube, r, U, 6, F, 6, D, 6, B, 6); break;
  default: error("rot face", f);
  }
  f *= 8;
  byte p;
  switch (r) {
  case 1:
    p         = cube[f+7];
    cube[f+7] = cube[f+5];
    cube[f+5] = cube[f+3];
    cube[f+3] = cube[f+1];
    cube[f+1] = p;
    p         = cube[f+6];
    cube[f+6] = cube[f+4];
    cube[f+4] = cube[f+2];
    cube[f+2] = cube[f];
    cube[f]   = p;
    break;

  case 2:
    p         = cube[f+1];
    cube[f+1] = cube[f+5];
    cube[f+5] = p;
    p         = cube[f+3];
    cube[f+3] = cube[f+7];
    cube[f+7] = p;
    p         = cube[f];
    cube[f]   = cube[f+4];
    cube[f+4] = p;
    p         = cube[f+2];
    cube[f+2] = cube[f+6];
    cube[f+6] = p;
    break;

  case 3:
    p         = cube[f+1];
    cube[f+1] = cube[f+3];
    cube[f+3] = cube[f+5];
    cube[f+5] = cube[f+7];
    cube[f+7] = p;
    p         = cube[f];
    cube[f]   = cube[f+2];
    cube[f+2] = cube[f+4];
    cube[f+4] = cube[f+6];
    cube[f+6] = p;
    break;

  default:
    error("rot", r);
  }
}

int idx_ic;
int idx_ie;
int idx_idx[NPIECE];
int idx_nc;
int idx_ne;
int idx;

void index_init() {
  idx_nc = 0;
  idx_ne = 0;
  idx = 0;
}

void index_last() {
  idx = ((idx>>2)<<1)|(idx&1);
}

#define FIND_CORNER(F0, O0, F1, O1, F2, O2, I0, I1, I2) \
  c0 = cube[POS(F0, O0)]; \
  if (c0 == f0) { \
    if (cube[POS(F1, O1)] == f1 && \
        cube[POS(F2, O2)] == f2) return I2; \
  } else if (c0 == f2) { \
    if (cube[POS(F1, O1)] == f0 && \
        cube[POS(F2, O2)] == f1) return I1; \
  } else if (c0 == f1) { \
    if (cube[POS(F1, O1)] == f2 && \
        cube[POS(F2, O2)] == f0) return I0; \
  }

int find_corner(byte &cube[], byte f0, byte f1, byte f2) {
  byte c0;
  FIND_CORNER(U, 2, B, 4, R, 2, 0,  1,  2);
  FIND_CORNER(U, 0, L, 0, B, 6, 3,  4,  5);
  FIND_CORNER(U, 6, F, 0, L, 2, 6,  7,  8);
  FIND_CORNER(U, 4, R, 0, F, 2, 9,  10, 11);
  FIND_CORNER(D, 0, L, 4, F, 6, 12, 13, 14);
  FIND_CORNER(D, 6, B, 0, L, 6, 15, 16, 17);
  FIND_CORNER(D, 4, R, 4, B, 2, 18, 19, 20);
  FIND_CORNER(D, 2, F, 4, R, 6, 21, 22, 23);
  return -1;
}

int index_corner(byte &cube[], byte f0, byte f1, byte f2) {
  int ic = find_corner(cube, f0, f1, f2);
  for (int i = 0; i < idx_nc; i++) {
    if (ic > idx_idx[i])
      ic -= 3;
  }
  idx = (idx*idx_ic) + ic;
  idx_idx[idx_nc++] = ic;
  idx_ic -= 3;
}

#define FIND_EDGE(F0, O0, F1, O1, I0, I1) \
  e0 = cube[POS(F0, O0)]; \
  if (e0 == f0) { \
    if (cube[POS(F1, O1)] == f1) return I1; \
  } else if (e0 == f1) { \
    if (cube[POS(F1, O1)] == f0) return I0; \
  }

int find_edge(byte &cube[], byte f0, byte f1) {
  byte e0;
  FIND_EDGE(U, 1, B, 5, 0,  1);
  FIND_EDGE(U, 7, L, 1, 2,  3);
  FIND_EDGE(U, 5, F, 1, 4,  5);
  FIND_EDGE(U, 3, R, 1, 6,  7);
  FIND_EDGE(L, 3, F, 7, 8,  9);
  FIND_EDGE(B, 7, L, 7, 10, 11);
  FIND_EDGE(D, 7, L, 5, 12, 13);
  FIND_EDGE(R, 3, B, 3, 14, 15);
  FIND_EDGE(D, 5, B, 1, 16, 17);
  FIND_EDGE(F, 3, R, 7, 18, 19);
  FIND_EDGE(D, 3, R, 5, 20, 21);
  FIND_EDGE(D, 1, F, 5, 22, 23);
  return -1;
}

void index_edge(byte &cube[], byte f0, byte f1) {
  int ie = find_edge(cube, f0, f1);
  for (int i = 0; i < idx_ne; i++) {
    if (ie > idx_idx[i])
      ie -= 2;
  }
  idx = (idx*idx_ie) + ie;
  idx_idx[idx_ne++] = ie;
  idx_ie -= 2;
}

bool valid_pieces(byte &cube[]) {
  return
    (find_edge(cube, U, F) >= 0) &&
    (find_edge(cube, U, L) >= 0) &&
    (find_edge(cube, U, B) >= 0) &&
    (find_edge(cube, U, R) >= 0) &&
    (find_edge(cube, F, L) >= 0) &&
    (find_edge(cube, L, B) >= 0) &&
    (find_edge(cube, B, R) >= 0) &&
    (find_edge(cube, R, F) >= 0) &&
    (find_edge(cube, D, F) >= 0) &&
    (find_edge(cube, D, L) >= 0) &&
    (find_edge(cube, D, B) >= 0) &&
    (find_edge(cube, D, R) >= 0) &&
    (find_corner(cube, U, F, L) >= 0) &&
    (find_corner(cube, U, L, B) >= 0) &&
    (find_corner(cube, U, B, R) >= 0) &&
    (find_corner(cube, U, R, F) >= 0) &&
    (find_corner(cube, D, F, R) >= 0) &&
    (find_corner(cube, D, R, B) >= 0) &&
    (find_corner(cube, D, B, L) >= 0) &&
    (find_corner(cube, D, L, F) >= 0);
}

void solve_phase(byte &cube[], int mtb, const byte &mtd[], bool dorot = true) {
  int sz = ArrayLen(mtd)/mtb;
  idx = sz-idx;
  if (idx > 0) {
    int i = (idx-1)*mtb;
    byte b = mtd[i++];
    if (b != 0xFF) {
      const int mvm = mtb*2-1;
      int mv = 0;
      int f0 = b / 3;
      int r0 = RFIX(b-(f0*3)+1);
      add_mv(f0, r0);
      if (dorot)
        rotate(cube, f0, r0);
      mv ++;
      while (mv < mvm) {
        b >>= 4;
        if ((mv & 1) != 0)
          b = mtd[i++];
        byte b0 = b & 0xF;
        if (b0 == 0xF)
          break;
        int f1 = b0 / 3;
        r0 = RFIX(b0-(f1*3)+1);
        if (f1 >= f0)
          f1 ++;
        f0 = f1;
        add_mv(f0, r0);
        if (dorot)
          rotate(cube, f0, r0);
        mv ++;
      }
    }
  }
}

void solve_one(byte &cube[], bool dorot) {
  mv_n = 0;

  idx_ic = 24;
  idx_ie = 24;

  index_init();
  index_edge(cube, D, F);
  index_edge(cube, D, R);
  solve_phase(cube, mtb0, mtd0);

  index_init();
  index_corner(cube, D, F, R);
  index_edge(cube, F, R);
  solve_phase(cube, mtb1, mtd1);

  index_init();
  index_edge(cube, D, B);
  solve_phase(cube, mtb2, mtd2);

  index_init();
  index_corner(cube, D, R, B);
  index_edge(cube, R, B);
  solve_phase(cube, mtb3, mtd3);

  index_init();
  index_edge(cube, D, L);
  solve_phase(cube, mtb4, mtd4);

  index_init();
  index_corner(cube, D, B, L);
  index_edge(cube, B, L);
  solve_phase(cube, mtb5, mtd5);

  index_init();
  index_corner(cube, D, L, F);
  index_edge(cube, L, F);
  solve_phase(cube, mtb6, mtd6);

  index_init();
  index_corner(cube, U, R, F);
  index_corner(cube, U, F, L);
  index_corner(cube, U, L, B);
  solve_phase(cube, mtb7, mtd7);

  index_init();
  index_edge(cube, U, R);
  index_edge(cube, U, F);
  index_edge(cube, U, L);
  index_last();
  solve_phase(cube, mtb8, mtd8, dorot);
}

#define MAP(UU, FF) (imap[((UU)*NFACE)+(FF)]*NFACE)

const byte imap[] = {
  /*       U   F   D   B   R   L
  /* U */ -1,  0, -1,  1,  2,  3,
  /* F */  4, -1,  5, -1,  6,  7,
  /* D */ -1,  8, -1,  9, 10, 11,
  /* B */ 12, -1, 13, -1, 14, 15,
  /* R */ 16, 17, 18, 19, -1, -1,
  /* L */ 20, 21, 22, 23, -1, -1
};

const byte fmap[] = {
  /* -- */
  /* UF */ U, F, D, B, R, L,
  /* -- */
  /* UB */ U, B, D, F, L, R,
  /* UR */ U, R, D, L, B, F,
  /* UL */ U, L, D, R, F, B,

  /* FU */ F, U, B, D, L, R,
  /* -- */
  /* FD */ F, D, B, U, R, L,
  /* -- */
  /* FR */ F, R, B, L, U, D,
  /* FL */ F, L, B, R, D, U,

  /* -- */
  /* DF */ D, F, U, B, L, R,
  /* -- */
  /* DB */ D, B, U, F, R, L,
  /* DR */ D, R, U, L, F, B,
  /* DL */ D, L, U, R, B, F,

  /* BU */ B, U, F, D, R, L,
  /* -- */
  /* BD */ B, D, F, U, L, R,
  /* -- */
  /* BR */ B, R, F, L, D, U,
  /* BL */ B, L, F, R, U, D,

  /* RU */ R, U, L, D, F, B,
  /* RF */ R, F, L, B, D, U,
  /* RD */ R, D, L, U, B, F,
  /* RB */ R, B, L, F, U, D,
  /* -- */
  /* -- */

  /* LU */ L, U, R, D, B, F,
  /* LF */ L, F, R, B, U, D,
  /* LD */ L, D, R, U, F, B,
  /* LB */ L, B, R, F, D, U
  /* -- */
  /* -- */
};

const byte omap[] = {
  /* -- */
  /* UF */ 0, 0, 0, 0, 0, 0,
  /* -- */
  /* UB */ 4, 4, 4, 4, 0, 0,
  /* UR */ 6, 0, 2, 4, 4, 0,
  /* UL */ 2, 0, 6, 4, 0, 4,

  /* FU */ 4, 4, 4, 4, 2, 6,
  /* -- */
  /* FD */ 0, 0, 0, 0, 6, 2,
  /* -- */
  /* FR */ 6, 6, 2, 6, 4, 0,
  /* FL */ 2, 2, 6, 2, 0, 4,

  /* -- */
  /* DF */ 4, 4, 4, 4, 4, 4,
  /* -- */
  /* DB */ 0, 0, 0, 0, 4, 4,
  /* DR */ 6, 4, 2, 0, 4, 0,
  /* DL */ 2, 4, 6, 0, 0, 4,

  /* BU */ 0, 0, 0, 0, 2, 6,
  /* -- */
  /* BD */ 4, 4, 4, 4, 6, 2,
  /* -- */
  /* BR */ 6, 2, 2, 2, 4, 0,
  /* BL */ 2, 6, 6, 6, 0, 4,

  /* RU */ 4, 2, 0, 6, 2, 2,
  /* RF */ 2, 2, 2, 6, 2, 2,
  /* RD */ 0, 2, 4, 6, 2, 2,
  /* RB */ 6, 2, 6, 6, 2, 2,
  /* -- */
  /* -- */

  /* LU */ 4, 6, 0, 2, 6, 6,
  /* LF */ 6, 6, 6, 2, 6, 6,
  /* LD */ 0, 6, 4, 2, 6, 6,
  /* LB */ 2, 6, 2, 2, 6, 6,
  /* -- */
  /* -- */
};

int  solve_n;
byte solve_fce[MV_MAX];
int  solve_rot[MV_MAX];
byte solve_cube[NFACE*8];

#ifdef SOLVE_PATTERN

void apply_pattern(byte &cube[], byte &pat_fce[], char &pat_rot[]) {
  // Add moves to transform solved cube into pattern
  for (int i = 0; i < ArrayLen(pat_fce); i++) {
    solve_fce[solve_n] = pat_fce[i];
    solve_rot[solve_n] = pat_rot[i];
    solve_n++;
  }

  // Apply moves in reverse to a solved position and
  // replace the scanned cube with position that if
  // "solved" will result in the pattern
  copy_cube(cube, solve_cube);
  for (int i = solve_n-1; i >= 0; i--)
    rotate(cube, solve_fce[i], -solve_rot[i]);
}

#endif

bool solve_nomap(byte &cube[]) {
  solve_n = 0;
  copy_cube(solve_cube, cube);
  solve_one(solve_cube, true);

  if (!solved(solve_cube))
    return false;

  // Record the solution
  solve_n = mv_n;
  for (int i = 0; i < mv_n; i++) {
    solve_fce[i] = mv_f[i];
    solve_rot[i] = mv_r[i];
  }

#ifdef SOLVE_PATTERN
  if (
#ifdef ONLY_PATTERNS
      // Only create patterns and never a normal solve
      true
#else
      // If the cube is solved, always create a pattern
      solve_n == 0 ||
      // Otherwise decide at random to occasionally create a pattern
      // rather than a normal solve
      Random(6) == 0
#endif
      ) {
    int pat = -1;
    while (pat < 0) {
      // Select one of the included patterns at random
      // (keep looping until one is selected and applied)
      pat = Random(5);
      switch (pat) {
#ifdef PATTERN_CHECKERBOARD
      case 0:
        apply_pattern(cube, checkerboard_fce, checkerboard_rot);
        break;
#endif
#ifdef PATTERN_SPOT
      case 1:
        apply_pattern(cube, spot_fce, spot_rot);
        break;
#endif
#ifdef PATTERN_SNAKE
      case 2:
        apply_pattern(cube, snake_fce, snake_rot);
        break;
#endif
#ifdef PATTERN_CUBES
      case 3:
        apply_pattern(cube, cubes_fce, cubes_rot);
        break;
#endif
#ifdef PATTERN_SUPERFLIP
      case 4:
        apply_pattern(cube, superflip_fce, superflip_rot);
        break;
#endif
      default:
        // Indicate that no pattern has been applied
        pat = -1;
        break;
      }
    }
  }
#endif

  return true;
}

byte pmap[NFACE];

void solve_remap(byte &cube[], int map) {
  for (int f = 0; f < NFACE; f++)
    pmap[fmap[map+f]] = f;

  for (int f = 0; f < NFACE; f++) {
    int fs = fmap[map+f]*8;
    int fd = f*8;
    int os = omap[map+f];
    for (int od = 0; od < 8; od++) {
      solve_cube[fd+od] = pmap[cube[fs+os]];
      os = (os+1)&7;
    }
  }

  solve_one(solve_cube, false);
  if (mv_n < solve_n) {
    solve_n = mv_n;
    for (int i = 0; i < mv_n; i++) {
      solve_fce[i] = fmap[map+mv_f[i]];
      solve_rot[i] = mv_r[i];
    }
  }
}

bool solve(byte &cube[]) {
  if (!solve_nomap(cube))
    return false;

  flash_blue();

  if (solve_n <= TARGET)
    return true;

  for (int i = 1; i < NFACE*4; i++) {
    solve_remap(cube, i*NFACE);
    if (solve_n <= TARGET)
      return true;
  }

  return true;
}

//-----------------------------------------------------------------------------
// High level routines to scan, solve and manipulate cube
//-----------------------------------------------------------------------------

byte clr_map[] = {0, 1, 2, 3, 4, 5};
int  clr_ord[NFACE*4];

void determine_colors(byte &cube[], const int t) {
  for (int i = 0; i < NFACE; i++)
    clr_ord[i] = 9*i+8;
  sort_colors(clr_ord, t, 1);
  for (int i = 0; i < NFACE; i++) {
    clr_ord[4*i+0] = 9*i+0;
    clr_ord[4*i+1] = 9*i+2;
    clr_ord[4*i+2] = 9*i+4;
    clr_ord[4*i+3] = 9*i+6;
  }
  sort_colors(clr_ord, t, 4);
  for (int i = 0; i < NFACE; i++) {
    clr_ord[4*i+0] = 9*i+1;
    clr_ord[4*i+1] = 9*i+3;
    clr_ord[4*i+2] = 9*i+5;
    clr_ord[4*i+3] = 9*i+7;
  }
  sort_colors(clr_ord, t, 4);
  for (int f = 0; f < NFACE; f++)
    clr_map[sc_clr[f*9+8]] = f;
  clr_chr[clr_map[CLR_R]] = CHR_R;
  clr_chr[clr_map[CLR_B]] = CHR_B;
  clr_chr[clr_map[CLR_O]] = CHR_O;
  clr_chr[clr_map[CLR_G]] = CHR_G;
  clr_chr[clr_map[CLR_W]] = CHR_W;
  clr_chr[clr_map[CLR_Y]] = CHR_Y;
  for (int f = 0; f < NFACE; f++)
    for (int o = 0; o < 8; o++)
      cube[POS(f, o)] = clr_map[sc_clr[f*9+o]];
}

int  uc = U;
int  fc = F;

int pieces_valid = 0;
int valid_i = 0;

bool scan_solve_cube(byte &cube[]) {
  flash_off();
  color_sensor_on();
  scan_face(1, 3, 2);
  tilt();
  scan_face(2, 2, 2);
  tilt();
  scan_face(3, 1, 2);
  tilt();
  scan_face(4, 0, 2);
  turn(-1);
  tilt();
  scan_face(5, 4, 6);
  tilt(2);
  scan_face(6, 5, 2);
  motors_off();
  flash_red();
  bool solved = false;
  ClearScreen();
  TextOut(0*6, LCD_LINE1, "Processing...");
  for (int i = 0; i < 6; i++) {
    determine_colors(cube, i);
    if (valid_pieces(cube)) {
      pieces_valid ++;
      valid_i = i;
      display_cube(cube);
      if (solve(cube)) {
        solved = true;
        break;
      }
    }
  }
  motors_on();
  ClearScreen();
  display_cube(cube);
  if (solved) {
    TextOut(0, LCD_LINE1, "Solving...");
    scan_away();
    flash_off();
    scan_wait();
  } else {
    TextOut(0, LCD_LINE1, "Scan error...");
    if (pieces_valid > 1) {
      TextOut(0, LCD_LINE1, "Cube");
      flash_blue();
    } else {
      flash_red();
    }
  }
  uc = L;
  fc = D;
  return solved;
}

void manipulate(byte &cube[], int f, int r, int rn) {
  int map = MAP(uc, fc);
  if (fmap[map+U] == f) {
    uc = fmap[map+D]; fc = fmap[map+B];
    Wait(50);
    tilt(2);
  } else if (fmap[map+F] == f) {
    uc = fmap[map+B]; fc = fmap[map+U];
    Wait(50);
    tilt();
  } else if (fmap[map+D] == f) {
    tilt_hold();
  } else if (fmap[map+B] == f) {
    uc = fmap[map+F]; fc = fmap[map+U];
    tilt_release();
    turn(2);
    tilt();
  } else if (fmap[map+R] == f) {
    uc = fmap[map+L]; fc = fmap[map+U];
    tilt_release();
    turn(1);
    tilt();
  } else { // fmap[map+L] == f
    uc = fmap[map+R]; fc = fmap[map+U];
    tilt_release();
    turn(-1);
    tilt();
  }
  Wait(150);
  rotate(cube, f, r);
  display_cube(cube);
  turn_face(-r, -rn);
}

//-----------------------------------------------------------------------------
// Main task controlling solve and calibration
//-----------------------------------------------------------------------------

int  recal = 0;

void calibrate_tilt() {
  if (recal <= 0) {
    // Calibrate tilt position at start and every 5 attempts
    // in case of tacho drift
    recal = 5;
    endstop(M_TILT);
    tilt_away();
  }
  recal --;
}

task main() {
  byte cube[NFACE*8];

  banner();
#ifdef NXT_8527
  TextOut(0, LCD_LINE3, "NXT (Orange box)");
#else
  TextOut(0, LCD_LINE3, "NXT 2.0");
#endif
  TextOut(0, LCD_LINE4, "By David Gilday");
  TextOut(0, LCD_LINE5, "mindcuber.com");

  // Read the white calibration value from the configuration file
  unsigned int cfgl;
  if (OpenFileRead(cfg_name, cfgl, cfgh) == LDR_SUCCESS) {
    Read(cfgh, white_rgb);
    CloseFile(cfgh);
  }

  start motor_task;
  flash_red();

  init_cube(cube);

  calibrate_tilt();
  endstop(M_SCAN);
  scan_away();
  scan_wait();
  motors_on();

  byte msg_line = LCD_LINE7;
  while (true) {
    ClearLine(msg_line);
    TextOut(0*6, msg_line, "Remove cube...");
    bool cal_white = false;
    while (!cube_absent()) {
      if (!cal_white && center_pressed()) {
        msg_line = LCD_LINE7;
        banner();
        TextOut(0*6, LCD_LINE4, "Calibrate white");
        TextOut(0*6, msg_line, "Remove cube...");
        cal_white = true;
      }
      Wait(1);
    }
    cube_detect_off();
    calibrate_tilt();
    flash_off();

    ClearLine(msg_line);
    TextOut(0*6, msg_line, "Insert cube...");
    while (!cube_present()) {
      if (left_pressed())
        move_rel(M_TURN, -T_ADJ);
      if (right_pressed())
        move_rel(M_TURN, T_ADJ);
      Wait(1);
    }
    cube_detect_off();

    banner();
    TextOut(0*6, LCD_LINE4, "Scanning...");

    create_log();

    bool solved = false;
    pieces_valid = 0;
    if (cal_white) {
      calibrate_white();
    } else {
      msg_line = LCD_LINE2;
      timer_start();
      for (int tries = 0; !solved && tries < 3; tries ++) {
        if (scan_solve_cube(cube)) {
          // Apply solution to the cube
          for (int i = 0; i < solve_n; i++) {
            int fi = solve_fce[i];
            int ri = solve_rot[i];
            int fo = opposite[fi];
            int rn = 0;
            for (int j = i+1; rn == 0 && j < solve_n; j++) {
              if (solve_fce[j] != fo)
                rn = solve_rot[j];
            }
            ClearLine(LCD_LINE2);
            TextOut(0*6, LCD_LINE2, "Move    of");
            NumOut(5*6,  LCD_LINE2, i+1);
            NumOut(11*6, LCD_LINE2, solve_n);
            manipulate(cube, fi, ri, rn);
          }
          timer_stop();
          flash_green();
          banner();
          TextOut(0*6, LCD_LINE3, "Solved!");
          TextOut(0*6, LCD_LINE4, "Moves");
          NumOut(6*6,  LCD_LINE4, solve_n);
          TextOut(0*6, LCD_LINE5, "Time");
          TextOut(6*6, LCD_LINE5, tmr_str);
          solved = true;
          msg_line = LCD_LINE7;
        }
      }
      timer_stop();
    }

    log(pieces_valid);
    log(valid_i);
    logeol();
    log(white_rgb[0]);
    log(white_rgb[1]);
    log(white_rgb[2]);
    logeol();
    close_log();

    scan_away();
    tilt_away();
    scan_wait();
    if (solved)
      turn(8);
  }
}

// END
