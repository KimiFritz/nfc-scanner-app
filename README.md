# nfc-scanner-app

Dieses Repository enthält die im 6. Semester des Moduls „Fallstudie“ an der Dualen Hochschule Baden-Württemberg Villingen-Schwenningen erarbeitete Lösung zur Entwicklung einer NFC-Scanner-App.

**Aufgabenstellung:** Ihre Aufgabe ist das Erstellen einer Android-App zum Lesen und Schreiben von NFC-Tags.

**Anforderungen:**  
- [X]  Die App unterstützt mindestens eine der beiden Plattformen Android oder iOS. → Android  
- [X]  Alle benötigten Berechtigungen müssen von der App angefordert werden.  
- [X]  Falls NFC im Gerät deaktiviert ist, muss ein Hinweis eingeblendet werden.  
- [X]  Während ein Lese- oder Schreibvorgang stattfindet, muss ein Hinweis mit einem “Abbrechen”-Button eingeblendet werden, sodass der Vorgang gestoppt werden kann.  
- [X]  Die Übersichtsseite besitzt einen Button, über den der Lesevorgang gestartet wird.  
- [X]  Die Übersichtsseite besitzt einen Button, über den der Schreibvorgang gestartet wird.  
- [X]  Es können beliebige Textinhalte in Form eines NDEF-Records auf einen NFC-Tag geschrieben werden.  
- [X]  Nach dem Lesen eines NFC-Tags werden folgende Informationen des NFC-Tags auf einer Detailseite angezeigt: `id`, `payload` und `isWritable`. `payload` ist der Wert des ersten NDEF-Records des NFC-Tags, falls vorhanden.  
- [X]  Auf der Detailseite kann die `payload` des NFC-Tags in die Zwischenablage kopiert werden.  
- [X]  Auf der Detailseite kann die `payload` des NFC-Tags geteilt werden.  
- [X]  Auf der Detailseite werden Bytes des NFC-Tags in Hexadezimal-Notation angezeigt.  

**Tipps:**  
- Texte und Dateien können über das Share-Plugin geteilt werden.  
- Es müssen nur NFC-Tags mit NDEF-Records vom Typ `Text` unterstützt werden.  
- Es existiert eine `NfcUtils`-Klasse (siehe [hier](https://capawesome.io/plugins/nfc/#utils)), die unterschiedliche, hilfreiche Methoden zur Verfügung stellt (z. B. das Erstellen eines NDEF-Records oder das Umwandeln von Bytes in Hexadezimal-Notation).  
- Für die Übersichtsseite kann ein einfaches Modal zum Einsatz kommen. Der eingescannte NFC-Tag kann als Parameter mitgegeben werden.  

**Plugins:**  
- [Share](https://capacitorjs.com/docs/apis/share)  
- [Clipboard](https://capacitorjs.com/docs/apis/clipboard)  
- [NFC](https://capawesome.io/plugins/nfc/)  

---

## Abgabe

- **Präsentation (PDF):** [abgabe/praesentation.pdf](./abgabe/praesentation.pdf)  
  *(Präsentation der App am 25.08.2025)*  

- **APK-Datei:** [abgabe/app-release.apk](./abgabe/app-release.apk)  
  *(installierbare Version der App für Android-Geräte)*  
