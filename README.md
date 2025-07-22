# nfc-scanner-app

Dieses Repository enthält die im 6. Semester des Moduls „Fallstudie“ an der Dualen Hochschule Baden-Württemberg Villingen-Schwenningen erarbeitete Lösung zur Entwicklung einer NFC-Scanner-App.

**Aufgabenstellung:** Ihre Aufgabe ist das Erstellen einer Android- oder iOS-App zum Lesen und Schreiben von NFC-Tags.

**Anforderungen:**  
- [ ]  Die App unterstützt mindestens eine der beiden Plattformen Android oder iOS.
- [ ]  Alle benötigten Berechtigungen müssen von der App angefordert werden.
- [ ]  Falls NFC im Gerät deaktiviert ist, muss ein Hinweis eingeblendet werden.
- [ ]  Während ein Lese- oder Schreibvorgang stattfindet, muss ein Hinweis mit einem “Abbrechen”-Button eingeblendet werden, sodass der Vorgang gestoppt werden kann.
- [ ]  Die Übersichtsseite besitzt einen Button, über den der Lesevorgang gestartet wird.
- [ ]  Die Übersichtsseite besitzt einen Button, über den der Schreibvorgang gestartet wird.
- [ ]  Es können beliebige Textinhalte in Form eines NDEF-Records auf einen NFC-Tag geschrieben werden.
- [ ]  Nach dem Lesen eines NFC-Tags werden folgende Informationen des NFC-Tags auf einer Detailseite angezeigt: `id`, `payload` und `isWritable`. `payload` ist der Wert des ersten NDEF-Records des NFC-Tags, falls vorhanden.
- [ ]  Auf der Detailseite kann die `payload` des NFC-Tags in die Zwischenablage kopiert werden.
- [ ]  Auf der Detailseite kann die `payload` des NFC-Tags geteilt werden.
- [ ]  Auf der Detailseite werden Bytes des NFC-Tags in Hexadezimal-Notation angezeigt.

**Tipps:**
- Texte und Dateien können über das Share-Plugin geteilt werden.
- Es müssen nur NFC-Tags mit NDEF-Records vom Typ `Text` unterstützt werden.
- Es existiert eine `NfcUtils`-Klasse (siehe [hier](https://capawesome.io/plugins/nfc/#utils)), die unterschiedliche, hilfreiche Methoden zur Verfügung stellt (z. B. das Erstellen eines NDEF-Records oder das Umwandeln von Bytes in Hexadezimal-Notation).
- Für die Übersichtsseite kann ein einfaches Modal zum Einsatz kommen. Der eingescannte NFC-Tag kann als Parameter mitgegeben werden.

**Plugins:**
- [Share](https://capacitorjs.com/docs/apis/share)
- [Clipboard](https://capacitorjs.com/docs/apis/clipboard)
- [NFC](https://capawesome.io/plugins/nfc/)

**Präsentation**
- [Google Slides](https://docs.google.com/presentation/d/1z23PDzNpM6THnzU-YDDMBi3YsSlLxQEGWBM_lDag4Qw/edit?usp=sharing)

**Fragen:**
- Sprache: Deutsch oder Englisch?
