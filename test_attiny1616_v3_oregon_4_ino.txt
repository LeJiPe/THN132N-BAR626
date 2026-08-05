/*
 * ================================================================
 *  PROGRAMME v3 - ATtiny1616 - Sonde Oregon THN132N (compat. BAR626)
 * ================================================================
 *
 * Cablage :
 *   - LED temoin       : PB1 (active LOW)
 *   - TMP117 SDA/SCL   : PA1 / PA2 (I2C, broches TWI0 par defaut)
 *   - TMP117 V+        : PA3 alimentation du TMP117 ~ 150mS
 *   - Dip switch canal : PC0 / PC1 / PC2 (pull-up interne)
 *   - RF433 TX (ASK)   : PB0
 *   - Quartz 32.768kHz : PB2 (TOSC1) / PB3 (TOSC2)
 *
 * Cycle de fonctionnement :
 *   1) Demarrage : lecture dip switch (canal + periode 39/41/43s),
 *      generation du rollcode (ID unique du chip), preparation de
 *      l'entete Oregon (ID capteur, canal, rollcode, niveau batterie).
 *   2) Veille STANDBY, reveil par RTC toutes les 39/41/43s (selon canal).
 *   3) A chaque reveil : lecture TMP117, encodage Oregon (temperature
 *      + checksum + CRC), emission RF433 (message envoye 2 fois,
 *      conformement au protocole v2.1), flash LED bref, retour veille.
 *
 * Origine du decodage Oregon (preambule/sync/encodage Manchester,
 * calcul checksum + CRC) : logique validee sur recepteur BAR626.
 * Portee ici sans modification de la logique de
 * calcul, uniquement adaptee aux broches et au flux de reveil/veille.
 * ================================================================
 */

#include <Arduino.h>
#include <avr/sleep.h>
#include <avr/interrupt.h>
#include <avr/cpufunc.h>
#include <SoftwareSerial.h>  // utile seulement pour debogage
#include <Wire.h>

// ---------- BROCHES ----------
#define LED_PIN         PIN_PB1   // active LOW
#define DIP_CANAL1_PIN  PIN_PC0
#define DIP_CANAL2_PIN  PIN_PC1
#define DIP_CANAL3_PIN  PIN_PC2
#define TMP117_VCC_PIN  PIN_PA3
#define RF433_TX_PIN    PIN_PB0
#define SOFTSERIAL_TX   PIN_PA5
#define SOFTSERIAL_RX   PIN_PA6

// ---------- I2C TMP117 ----------
#define TMP117_I2C_ADDR   0x48   // adresse par defaut (ADD0 relie a GND)
#define TMP117_REG_TEMP   0x00

// ---------- RTC ----------
// Prescaler /64 -> 32768/64 = 512 Hz. Couvre la plage complete
// 39s (canal normal) a 86s (canal economie), compteur 16 bits
// (max 65535) : 86s x 512Hz = 44032, confortable.
#define RTC_FREQ_HZ   512UL

// ---------- TIMING OREGON v2.1 (RF433 OOK/ASK) ----------
// Valeurs NOMINALES (protocole). Elles sont corrigees au demarrage
// par calibrerTimingViaQuartz() pour compenser l'imprecision propre
// a chaque exemplaire de l'oscillateur interne 20 MHz (non pilote
// par quartz, tolerance usine typique +/-1 a 3%).
const unsigned long OREGON_TIME_NOMINAL    = 480;  // ~490 us
const unsigned long OREGON_TWOTIME_NOMINAL = 969;  // ~980 us

// Valeurs effectivement utilisees par sendZero()/sendOne(), calculees
// une fois au demarrage a partir du facteur de correction mesure.
unsigned long oregonTimeCorrige    = OREGON_TIME_NOMINAL;
unsigned long oregonTwoTimeCorrige = OREGON_TWOTIME_NOMINAL;

#define SEND_HIGH()  digitalWrite(RF433_TX_PIN, HIGH)
#define SEND_LOW()   digitalWrite(RF433_TX_PIN, LOW)

SoftwareSerial debugSerie(SOFTSERIAL_RX, SOFTSERIAL_TX); // (RX, TX)

volatile bool reveilRTC = false;

uint8_t canalSelectionne = 1;       // 1, 2 ou 3 (choix utilisateur)
uint8_t canalOregon = 0x1;          // valeur protocole Oregon (0x1/0x2/0x4)
unsigned long periodeSecondes = 39;
bool modeEconomie = false;          // true = periode doublee (1 combinaison sur 2)
uint8_t rollCode = 0;
const uint8_t BATT_LEVEL = 0x08;    // 0x08 = batterie OK (pas de mesure batterie pour l'instant)
unsigned int compteurCycles = 0;

// Message Oregon (8 octets) + buffer intermediaire pour le CRC
byte OregonMessageBuffer[8];
byte OregonCRC2Buffer[4];

// ================================================================
//  INTERRUPTION RTC
// ================================================================
ISR(RTC_CNT_vect) {
  RTC.INTFLAGS = RTC_OVF_bm;
  reveilRTC = true;
}

// ================================================================
//  DIP SWITCH DE CANAL
// ================================================================
void lireCanalDipSwitch() {
  pinMode(DIP_CANAL1_PIN, INPUT_PULLUP);
  pinMode(DIP_CANAL2_PIN, INPUT_PULLUP);
  pinMode(DIP_CANAL3_PIN, INPUT_PULLUP);
  delay(10);

  bool sw1 = (digitalRead(DIP_CANAL1_PIN) == LOW); // ferme = actif
  bool sw2 = (digitalRead(DIP_CANAL2_PIN) == LOW);
  bool sw3 = (digitalRead(DIP_CANAL3_PIN) == LOW);
  uint8_t nbFermes = sw1 + sw2 + sw3;

  modeEconomie = false;

  if (nbFermes == 1) {
    // Mode normal : un seul switch ferme designe le canal
    if (sw1) {
      canalSelectionne = 1; periodeSecondes = 39; canalOregon = 0x1;
    } else if (sw2) {
      canalSelectionne = 2; periodeSecondes = 41; canalOregon = 0x2;
    } else {
      canalSelectionne = 3; periodeSecondes = 43; canalOregon = 0x4;
    }
  } else if (nbFermes == 2) {
    // Mode economie : combinaison inversee (un seul switch reste
    // OUVERT designe le canal). Periode doublee : le RTC se reveille
    // deux fois moins souvent (78/82/86s au lieu de 39/41/43s), ce
    // qui economise aussi l'energie du reveil et de la lecture TMP117,
    // pas seulement celle de l'emission RF.
    modeEconomie = true;
    if (!sw1) {
      canalSelectionne = 1; periodeSecondes = 78; canalOregon = 0x1;
    } else if (!sw2) {
      canalSelectionne = 2; periodeSecondes = 82; canalOregon = 0x2;
    } else {
      canalSelectionne = 3; periodeSecondes = 86; canalOregon = 0x4;
    }
  } else {
    debugSerie.println(F("Combinaison dip switch invalide (0 ou 3 fermes) -> canal 1 normal par defaut"));
    canalSelectionne = 1; periodeSecondes = 39; canalOregon = 0x1;
  }
}

// ================================================================
//  ROLLCODE (a partir du numero de serie unique du chip)
// ================================================================
uint8_t genererRollCode() {
  uint8_t code = 0;
  volatile uint8_t *sernum = &SIGROW.SERNUM0; // 10 octets consecutifs
  for (uint8_t i = 0; i < 10; i++) {
    code ^= sernum[i];
  }
  return code;
}

// ================================================================
//  QUARTZ 32.768 kHz (TOSC1/TOSC2 = PB2/PB3)
// ================================================================
void configurerQuartz32k() {
  _PROTECTED_WRITE(CLKCTRL.XOSC32KCTRLA, 0x00);
  while (CLKCTRL.MCLKSTATUS & CLKCTRL_XOSC32KS_bm) {}

  uint8_t config = CLKCTRL_CSUT_32K_gc;
  _PROTECTED_WRITE(CLKCTRL.XOSC32KCTRLA, config);

  config |= CLKCTRL_RUNSTDBY_bm | CLKCTRL_ENABLE_bm;
  _PROTECTED_WRITE(CLKCTRL.XOSC32KCTRLA, config);
}

// ================================================================
//  RTC (periode dynamique selon canal)
// ================================================================
void configurerRTC(unsigned long periodeS) {
  while (RTC.STATUS > 0) {}

  RTC.CLKSEL = RTC_CLKSEL_XOSC32K_gc;

  unsigned long cycles = periodeS * RTC_FREQ_HZ;
  if (cycles > 65536UL) cycles = 65536UL;
  uint16_t perValue = (uint16_t)(cycles - 1);

  while (RTC.STATUS & RTC_PERBUSY_bm) {}
  RTC.PER = perValue;

  RTC.INTCTRL = RTC_OVF_bm;

  while (RTC.STATUS & RTC_CTRLABUSY_bm) {}
  RTC.CTRLA = RTC_PRESCALER_DIV64_gc
            | RTC_RTCEN_bm
            | RTC_RUNSTDBY_bm;
}

// ================================================================
//  LED (active LOW)
// ================================================================
void ledOn()  { digitalWrite(LED_PIN, LOW); }
void ledOff() { digitalWrite(LED_PIN, HIGH); }

void flashLed() {
  ledOn();
  delay(50);
  ledOff();
}

// ================================================================
//  VEILLE (STANDBY - seul mode compatible reveil RTC sur ATtiny1616)
// ================================================================
void entrerEnVeille() {
  ADC0.CTRLA &= ~ADC_ENABLE_bm;

  set_sleep_mode(SLEEP_MODE_STANDBY);

  cli();
  sleep_enable();
  sei();
  sleep_cpu();
  sleep_disable();

  ADC0.CTRLA |= ADC_ENABLE_bm;
}

// ================================================================
//  LECTURE TMP117 (I2C direct, sans librairie Adafruit)
// ================================================================
float lireTMP117() {
  Wire.beginTransmission(TMP117_I2C_ADDR);
  Wire.write(TMP117_REG_TEMP);
  Wire.endTransmission(false);

  Wire.requestFrom(TMP117_I2C_ADDR, (uint8_t)2);
  if (Wire.available() < 2) {
    debugSerie.println(F("Erreur lecture TMP117"));
    return NAN;
  }

  int16_t brut = (Wire.read() << 8) | Wire.read();

  return brut * 0.0078125f; // resolution TMP117 : 1 LSB = 0.0078125 degC
}

// ================================================================
//  PROTOCOLE OREGON v2.1 - ENCODAGE MANCHESTER + ENVOI RF
// ================================================================

inline void sendZero(void) {
  SEND_HIGH();
  delayMicroseconds(oregonTimeCorrige);
  SEND_LOW();
  delayMicroseconds(oregonTwoTimeCorrige);
  SEND_HIGH();
  delayMicroseconds(oregonTimeCorrige);
}

inline void sendOne(void) {
  SEND_LOW();
  delayMicroseconds(oregonTimeCorrige);
  SEND_HIGH();
  delayMicroseconds(oregonTwoTimeCorrige);
  SEND_LOW();
  delayMicroseconds(oregonTimeCorrige);
}

inline void sendQuarterMSB(const byte data) {
  (bitRead(data, 4)) ? sendOne() : sendZero();
  (bitRead(data, 5)) ? sendOne() : sendZero();
  (bitRead(data, 6)) ? sendOne() : sendZero();
  (bitRead(data, 7)) ? sendOne() : sendZero();
}

inline void sendQuarterLSB(const byte data) {
  (bitRead(data, 0)) ? sendOne() : sendZero();
  (bitRead(data, 1)) ? sendOne() : sendZero();
  (bitRead(data, 2)) ? sendOne() : sendZero();
  (bitRead(data, 3)) ? sendOne() : sendZero();
}

void sendData(byte *data, byte size) {
  for (byte i = 0; i < size; ++i) {
    sendQuarterMSB(data[i]);
    sendQuarterLSB(data[i]);
  }
}

inline void sendPreamble(void) {
  byte PREAMBLE[] = {0xFF, 0xFF};
  sendData(PREAMBLE, 2);
}

inline void sendSync(void) {
  sendQuarterLSB(0xA);
}

void sendOregon(byte *data, byte size) {
  sendPreamble();
  sendSync();
  sendData(data, size);
}

// Renseigne une fois pour toutes le canal, le rollcode et le niveau
// batterie dans l'entete du message (octets 2 et 3)
void setChannelRoll(byte *data, byte channel, byte rc, byte battLevel) {
  data[2] = (channel << 4) + (rc >> 4);
  OregonCRC2Buffer[1] = (channel << 4);

  data[3] = (rc << 4) + battLevel;
  OregonCRC2Buffer[1] += battLevel;
}

// Encode la temperature (BCD) dans les octets 4 et 5 du message
void setTemperature(byte *data, float temp) {
  data[5] = 0x00;
  OregonCRC2Buffer[3] = 0x00;

  if (temp < 0) {
    data[5] = 0x08;
    OregonCRC2Buffer[3] = 0x08;
    temp *= -1;
  }

  int tempInt = (int)temp;
  int td = (int)(tempInt / 10);                                   // dizaines
  int tf = (int)round(((float)tempInt / 10 - (float)td) * 10);    // unites
  int tempFloat = (int)round((temp - (float)tempInt) * 10);       // dixiemes

  data[5] += (td << 4);
  data[4] = (tempFloat << 4) + tf;

  OregonCRC2Buffer[2] = (tempFloat << 4) + tf;
  OregonCRC2Buffer[3] += (td << 4);
}

int Sum(byte count, const byte *data) {
  int s = 0;
  for (byte i = 0; i < 6; i++) {
    s += (data[i] & 0xF0) >> 4;
    s += (data[i] & 0xF);
  }
  if (int(count) != count) {
    s += (data[count] & 0xF0) >> 4;
  }
  return s;
}

// Checksum + CRC final (logique validee par l'auteur avec son
// BAR626 - conservee telle quelle)
void calculateAndSetChecksum(byte *data) {
  int s = Sum(6, data);
  data[6] = ((s & 0x0F) << 4) + ((s & 0xF0) >> 4);

  uint8_t crc = 0x4A; // valeur d'initialisation
  for (size_t i = 1; i < 4; i++) {
    crc ^= OregonCRC2Buffer[i];
    for (uint8_t j = 0; j < 8; j++) {
      if (crc & 0x80) {
        crc = (crc << 1) ^ 0x07; // polynome
      } else {
        crc <<= 1;
      }
    }
  }
  data[7] = ((crc ^ 0x59) << 4) + (((crc ^ 0x59) >> 4) & 0x0F);// Xor final iversion quartet
}

// Assemble, affiche (debug) et envoie le message complet pour une
// temperature donnee
void envoyerMessageOregon(float temperature) {
  setTemperature(OregonMessageBuffer, temperature);
  calculateAndSetChecksum(OregonMessageBuffer);

  debugSerie.print(F("Trame Oregon : "));
  for (byte i = 0; i < sizeof(OregonMessageBuffer); ++i) {
    debugSerie.print(OregonMessageBuffer[i] >> 4, HEX);
    debugSerie.print(OregonMessageBuffer[i] & 0x0F, HEX);
  }
  debugSerie.println();

  sendOregon(OregonMessageBuffer, 8);
  SEND_LOW(); // ligne indispensable : force l'etat bas entre les deux trames

  delay(10); // pause entre les deux emissions - timing non critique (+/- 1ms)

  sendOregon(OregonMessageBuffer, 8);     // le protocole v2.1 envoie 2 fois

  SEND_LOW();
}

// ================================================================
//  CALIBRATION DU TIMING RF VIA LE QUARTZ (RTC)
// ================================================================
// Mesure l'ecart reel de l'oscillateur interne 20 MHz par rapport
// au nominal, en comparant micros() (base sur cet oscillateur
// imprecis) a une duree de exactement 1.000000 s mesuree par le RTC
// (base sur le quartz 32.768 kHz, precis). Le facteur obtenu sert a
// corriger les delais utilises pour l'emission RF (sendZero/sendOne).
//
// Prerequis : le RTC doit deja etre configure et actif (fait dans
// configurerRTC()), et les interruptions globales activees (sei())
// car micros() en depend.
void calibrerTimingViaQuartz() {
  // Frequence RTC utilisee ici = RTC_FREQ_HZ (1024 Hz, prescaler /32)
  // 1024 cycles RTC = exactement 1.000000 s (reference quartz)
  const uint16_t CYCLES_UNE_SECONDE = (uint16_t)RTC_FREQ_HZ;
  const unsigned long PAS_US = 1000; // pas de mesure : 1 ms

  // Attend que le RTC ait REELLEMENT commence a avancer avant de
  // demarrer la mesure : le temps de demarrage du quartz (CSUT,
  // configure volontairement large dans configurerQuartz32k) laisse
  // RTC.CNT fige un moment apres l'activation - si on ne l'exclut
  // pas, ce temps mort fausse completement la calibration.
  uint16_t valInitiale = RTC.CNT;
  while (RTC.CNT == valInitiale) {
    // attente active jusqu'au premier incrementation reelle du RTC
  }

  uint16_t depart = RTC.CNT;
  unsigned long microsDemandes = 0;

  // Tant qu'une seconde reelle (quartz) ne s'est pas ecoulee, on
  // demande des pas de 1000us via delayMicroseconds() - EXACTEMENT
  // le meme mecanisme que celui utilise par sendZero()/sendOne().
  // On compte combien de microsecondes "demandees" ont ete
  // necessaires pour couvrir cette seconde reelle.
  while ((uint16_t)(RTC.CNT - depart) < CYCLES_UNE_SECONDE) {
    delayMicroseconds(PAS_US);
    microsDemandes += PAS_US;
  }

  // Duree reelle EFFECTIVEMENT ecoulee (peut legerement depasser
  // CYCLES_UNE_SECONDE a cause du pas de mesure) - on l'utilise
  // telle quelle plutot que de supposer exactement 1.000000s, ce
  // qui elimine le biais introduit par le depassement de boucle.
  uint16_t cyclesReels = (uint16_t)(RTC.CNT - depart);
  float secondesReelles = (float)cyclesReels / (float)RTC_FREQ_HZ;

  // facteur = rapport (temps nominal demande) / (temps reel ecoule)
  float facteurBrut = (float)microsDemandes / (secondesReelles * 1000000.0f);

  // Constante d'ajustement fixe : compense un biais systematique de
  // cette methode de mesure (delayMicroseconds() a ~1000us n'a pas
  // exactement le meme comportement qu'a ~470-960us). Determinee une
  // fois pour toutes par comparaison directe au scope entre la trame
  // emulee et celle d'une sonde Oregon d'origine (valeurs cibles
  // validees : 477us / 965us). A conserver telle quelle tant que le
  // code de sendZero()/sendOne() ne change pas.
  const float CORRECTION_REFERENCE = 1.00893f;
  float facteur = facteurBrut * CORRECTION_REFERENCE;

  oregonTimeCorrige    = (unsigned long)(OREGON_TIME_NOMINAL    * facteur + 0.5f);
  oregonTwoTimeCorrige = (unsigned long)(OREGON_TWOTIME_NOMINAL * facteur + 0.5f);

  debugSerie.print(F("Calibration : us demandes = "));
  debugSerie.print(microsDemandes);
  debugSerie.print(F(" | duree reelle mesuree = "));
  debugSerie.print(secondesReelles, 6);
  debugSerie.println(F(" s"));
  debugSerie.print(F("Facteur brut (mesure RTC) = "));
  debugSerie.println(facteurBrut, 5);
  debugSerie.print(F("Facteur de correction (ajuste) = "));
  debugSerie.println(facteur, 5);
  debugSerie.print(F("OREGON_TIME corrige = "));
  debugSerie.print(oregonTimeCorrige);
  debugSerie.print(F(" us | OREGON_TWOTIME corrige = "));
  debugSerie.print(oregonTwoTimeCorrige);
  debugSerie.println(F(" us"));
}

// ================================================================
//  SETUP
// ================================================================
void setup() {
  pinMode(LED_PIN, OUTPUT);
  ledOff();

  pinMode(TMP117_VCC_PIN, OUTPUT);
  digitalWrite(TMP117_VCC_PIN, LOW); // eteint par defaut, alimente uniquement le temps de la mesure

  pinMode(RF433_TX_PIN, OUTPUT);
  SEND_LOW();

  debugSerie.begin(9600);
  delay(200);
  debugSerie.println(F("=== Sonde Oregon THN132N - demarrage ==="));

  Wire.pins(PIN_PA1, PIN_PA2); // force SDA=PA1/SCL=PA2 (necessaire sur ce variant de core)
  Wire.begin();

  lireCanalDipSwitch();
  rollCode = genererRollCode();

  debugSerie.print(F("Canal : "));
  debugSerie.print(canalSelectionne);
  debugSerie.print(F(" | Mode : "));
  debugSerie.print(modeEconomie ? F("ECONOMIE") : F("normal"));
  debugSerie.print(F(" | Periode : "));
  debugSerie.print(periodeSecondes);
  debugSerie.print(F("s | Rollcode : 0x"));
  debugSerie.println(rollCode, HEX);

  // Entete du message (ID capteur THN132N, canal, rollcode, batterie)
  OregonMessageBuffer[0] = 0xEC;
  OregonMessageBuffer[1] = 0x40;
  setChannelRoll(OregonMessageBuffer, canalOregon, rollCode, BATT_LEVEL);

  configurerQuartz32k();
  configurerRTC(periodeSecondes);

  sei();

  calibrerTimingViaQuartz();

  debugSerie.println(F("RTC configure. Entree en veille..."));

  //##########################################################################
    delay(1000);
    digitalWrite(TMP117_VCC_PIN, HIGH); // mise sous tension TMP117 150ms avant la mesure
    flashLed();
    delay(120); // + flashLed() ~30ms 40ms de marge (min. mesure 110ms)
    float temperature = lireTMP117();
    digitalWrite(TMP117_VCC_PIN, LOW);  // coupure TMP117

    debugSerie.print(F("Reveil #"));
    debugSerie.print(compteurCycles);
    debugSerie.print(F(" | Temp = "));
    debugSerie.print(temperature);
    debugSerie.println(F(" C"));

    if (!isnan(temperature)) {
      envoyerMessageOregon(temperature);
    } else {
      debugSerie.println(F("Temperature invalide, envoi annule"));
    }


  //############################################################################

  entrerEnVeille();
}

// ================================================================
//  LOOP
// ================================================================
void loop() {
  if (reveilRTC) {
    reveilRTC = false;
    compteurCycles++;

    digitalWrite(TMP117_VCC_PIN, HIGH); // mise sous tension TMP117 150ms avant la mesure
    flashLed();
    delay(120); // + flashLed() ~30ms 40ms de marge (min. mesure 110ms)
    float temperature = lireTMP117();

    digitalWrite(TMP117_VCC_PIN, LOW);  // coupure TMP117

    debugSerie.print(F("Reveil #"));
    debugSerie.print(compteurCycles);
    debugSerie.print(F(" | Temp = "));
    debugSerie.print(temperature);
    debugSerie.println(F(" C"));

    if (!isnan(temperature)) {
      envoyerMessageOregon(temperature);
    } else {
      debugSerie.println(F("Temperature invalide, envoi annule"));
    }

    //flashLed();
    debugSerie.println(F("Retour en veille"));
  }

  entrerEnVeille();
}
