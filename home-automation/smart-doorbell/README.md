# DIY Smart Doorbell (Analog to Digital)

Dit project is een modificatie van een traditionele 'domme' deurbel naar een slim IoT-apparaat, direct geïntegreerd in Home Assistant. Het fungeert als de brug tussen een analoog stroomcircuit en het digitale netwerk.

Dit project bouwt voort op de bekende hardware-architectuur uit [de tutorial van Frenck](https://frenck.dev/diy-smart-doorbell-for-just-2-dollar/), maar de ESPHome-code is specifiek herschreven en geoptimaliseerd voor **mechanische deurbellen**.

---

## Hardware & Architectuur
* **Microcontroller:** ESP8266 (ESP-01S module).
* **Actuator:** ESP-01 Relay Module (om het fysieke stroomcircuit van de gong te sluiten).
* **Input:** De originele, fysieke deurbelknop bij de voordeur (verbonden met GPIO2).
* **Firmware:** ESPHome.

---

## Waarom de originele code is aangepast

Tijdens de implementatie bleek de standaard logica uit de tutorial niet geschikt voor mijn specifieke hardware (een mechanische deurbel met een fysieke 'ding-dong' klepel). 

Ik heb de volgende drie architectonische wijzigingen doorgevoerd in de YAML-configuratie:

### 1. Vaste Puls-tijd vs. Release-trigger (De belangrijkste fix)
* **Het probleem:** De originele code vuurt het relais af bij `on_press` en stopt het direct bij `on_release`. Bij een mechanische deurbel zorgt een (te) korte druk van de postbode ervoor dat de fysieke klepel zijn cyclus niet kan afmaken (een halve 'ding' zonder 'dong').
* **De oplossing:** Ik heb de `on_release` trigger volledig verwijderd. In plaats daarvan forceert mijn code nu een vaste stroompuls van exact 500ms bij een druk op de knop, ongeacht hoe snel de bezoeker de knop weer loslaat. Dit garandeert altijd een volledige, heldere beltoon.

```yaml
on_press:
  then:
    if:
      condition:
        - switch.is_on: chime_active
      then:
        - switch.turn_on: relay
        - delay: 500ms  # Vaste tijdsduur voor een mechanische cyclus
        - switch.turn_off: relay
```

### 2.GPIO Open-Drain Configuratie
Het gebruikte ESP-01 relaisbordje luistert soms nauw qua voltages. Om zwevende statussen ("floating pins") en ongewenst rammelen van het relais te voorkomen, is de relais-pin expliciet geconfigureerd met open_drain: true.

### 3. Netwerk Redundantie
Omdat een deurbel een kritiek onderdeel is van de woning, heb ik een fallback Wi-Fi hotspot en Captive Portal geconfigureerd. Mocht de OPNsense router onverhoopt uitvallen, zendt de deurbel zijn eigen "Deurbel" netwerk uit, zodat beheer en updates mogelijk blijven zonder de fysieke behuizing te moeten demonteren.

De "Chime Active" Vlag
Net als in de tutorial, is er een globale Boolean variabele (chime_active) geconfigureerd. Dit creëert een virtuele schakelaar in Home Assistant waarmee de fysieke gong softwarematig kan worden uitgezet (bijvoorbeeld als de baby slaapt). De deurbelknop registreert dan nog steeds de beweging en stuurt een push-notificatie naar mijn telefoon, maar het relais naar de gong wordt geblokkeerd.

De Automatisering (Node-RED Logica)
Naast de hardware-aansturing op de ESP-01, wordt het drukken op de deurbel via Home Assistant afgevangen in Node-RED. Dit triggert een flow met de volgende logica:

Rate Limiting (Anti-Spam): Een delay-node beperkt de input tot maximaal 1 trigger per 30 seconden. Dit voorkomt audio-spam wanneer iemand de knop herhaaldelijk indrukt.

Conditionele Routering (Stille Modus): De flow controleert direct de status van input_boolean.slaaptijd. Als de baby slaapt, wordt de actie omgeleid: de audio-notificaties in huis worden geblokkeerd en er wordt uitsluitend een stille push-notificatie naar de telefoons gestuurd.

State Management (Volume Snapshotting): Als de audio-notificatie wél is toegestaan, leest Node-RED eerst het huidige volume van de Google Nest Hubs uit en slaat dit op in tijdelijke variabelen (JSONata expressies). De speakers worden naar 90% gezet voor de beltoon, waarna het originele volume exact wordt hersteld.

Cross-Device Acties: Bij een deurbel-trigger wordt de status van de Android TV in de woonkamer uitgelezen. Als de status playing is, wordt de media automatisch gepauzeerd zodat de deurbel (en de bezoeker) niet overstemd worden.
