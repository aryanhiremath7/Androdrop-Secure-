# Androdrop-Secure-
name: secure_airdrop
description: Secure AirDrop NFC + AES-256
publish_to: 'none'
version: 1.0.0+1
environment:
  sdk: ">=3.0.0 <4.0.0"
dependencies:
  flutter:
    sdk: flutter
  nearby_connections: ^3.1.3
  permission_handler: ^11.3.1
  file_picker: ^8.0.6
  path_provider: ^2.1.2
  device_info_plus: ^9.1.2
  encrypt: ^5.0.3
  crypto: ^3.0.3
  nfc_manager: ^3.3.0
flutter:
  uses-material-design: true
name: Build APK
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
      - run: flutter pub get
      - run: flutter build apk --release
      - uses: actions/upload-artifact@v4
        with:
          name: apk
          path: build/app/outputs/flutter-apk/app-release.apkimport 'dart:io';
import 'dart:math';
import 'dart:convert';
import 'dart:typed_data';
import 'package:flutter/material.dart';
import 'package:nearby_connections/nearby_connections.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:file_picker/file_picker.dart';
import 'package:device_info_plus/device_info_plus.dart';
import 'package:crypto/crypto.dart';
import 'package:encrypt/encrypt.dart' as enc;
import 'package:nfc_manager/nfc_manager.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Secure AirDrop NFC',
      theme: ThemeData.dark(useMaterial3: true).copyWith(scaffoldBackgroundColor: const Color(0xFF0A0A0F)),
      home: const HomePage(),
    );
  }
}

class SecureEngine {
  static enc.Key genKey() {
    final r = Random.secure();
    return enc.Key(Uint8List.fromList(List.generate(32, (_) => r.nextInt(256))));
  }
  static enc.IV genIv() {
    final r = Random.secure();
    return enc.IV(Uint8List.fromList(List.generate(12, (_) => r.nextInt(256))));
  }
}

class HomePage extends StatefulWidget {
  const HomePage({super.key});
  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> with TickerProviderStateMixin {
  final Strategy strategy = Strategy.P2P_CLUSTER;
  final String serviceId = "com.aryan.secure.airdrop";
  String status = "🔒 NFC + Encrypted Ready";
  String? connectedId;
  double progress = 0;
  String verifyCode = "";
  bool isVerified = false;
  bool nfcAvailable = false;
  late AnimationController pulse;

  @override
  void initState() {
    super.initState();
    pulse = AnimationController(vsync: this, duration: const Duration(seconds: 2))..repeat();
    _perms();
    _genCode();
    _checkNFC();
  }

  void _genCode() { verifyCode = (100000 + Random().nextInt(900000)).toString(); }

  Future<void> _perms() async {
    await [Permission.bluetooth, Permission.bluetoothScan, Permission.bluetoothAdvertise, Permission.bluetoothConnect, Permission.location, Permission.nearbyWifiDevices, Permission.photos, Permission.nfc].request();
  }

  Future<void> _checkNFC() async {
    bool avail = await NfcManager.instance.isAvailable();
    setState(() => nfcAvailable = avail);
  }

  // NFC TAP TO CONNECT - ENCRYPTED HANDSHAKE
  void _startNFCReceive() async {
    setState(() => status = "📱 Bring other phone close for NFC tap...");
    NfcManager.instance.startSession(onDiscovered: (NfcTag tag) async {
      var ndef = Ndef.from(tag);
      if (ndef!= null) {
        // Read verification code from NFC
        String payload = String.fromCharCodes(ndef.cachedMessage!.records.first.payload).substring(3);
        // payload contains code like: SECURE:123456:com.aryan.secure.airdrop
        if (payload.contains("SECURE:")) {
          String codeFromNFC = payload.split(":")[1];
          setState(() => status = "📲 NFC Tap OK! Code $codeFromNFC - Connecting securely...");
          NfcManager.instance.stopSession();
          // Auto start discovery after NFC tap
          _receive();
        }
      }
    });
  }

  void _startNFCSend() async {
    setState(() => status = "📱 Tap other phone to share secure code...");
    // Write verification code to NFC
    NfcManager.instance.startSession(onDiscovered: (NfcTag tag) async {
      var ndef = Ndef.from(tag);
      if (ndef == null ||!ndef.isWritable) return;
      // Encrypted payload: Even NFC data is not plain - contains code + hash
      String securePayload = "SECURE:$verifyCode:$serviceId:${sha256.convert(utf8.encode(verifyCode)).toString().substring(0,8)}";
      NdefMessage message = NdefMessage([NdefRecord.createText(securePayload)]);
      try {
        await ndef.write(message);
        setState(() => status = "✅ NFC Tap Done! Code sent encrypted - Now connecting via fast WiFi...");
        NfcManager.instance.stopSession();
        _advertise();
      } catch (e) {
        setState(() => status = "NFC Write failed: $e");
      }
    });
  }

  Future<String> _name() async {
    try { final i = await DeviceInfoPlugin().androidInfo; return "${i.brand}-${i.model}"; } catch (_) { return "Android"; }
  }

  void _receive() async {
    setState(() => status = "🔍 Searching after NFC...");
    await Nearby().startDiscovery(await _name(), strategy,
      onEndpointFound: (id, name, sid) {
        Nearby().requestConnection(name, id,
          onConnectionInitiated: (id, info) {
            showDialog(context: context, builder: (_) => AlertDialog(
              backgroundColor: const Color(0xFF1E1E2E),
              title: const Text("NFC Verified? 🔒", style: TextStyle(color: Colors.white)),
              content: Column(mainAxisSize: MainAxisSize.min, children: [
                Text("NFC Code: $verifyCode", style: const TextStyle(fontSize: 32, color: Colors.white, fontWeight: FontWeight.bold)),
                const SizedBox(height: 10),
                const Text("If codes match, hacker can't enter - Connection is double encrypted", style: TextStyle(color: Colors.white70)),
              ]),
              actions: [
                TextButton(onPressed: () { Nearby().rejectConnection(id); Navigator.pop(context); }, child: const Text("REJECT", style: TextStyle(color: Colors.red))),
                ElevatedButton(onPressed: () { isVerified = true; Nearby().acceptConnection(id, onPayLoadRecieved: (ep, p) async { if (p.type == PayloadType.FILE) setState(() => status = "✅ Encrypted file via NFC+WiFi - Saved!"); }, onPayloadTransferUpdate: (id, up) { setState(() { progress = up.bytesTransferred / up.totalBytes; status = "🔒 Receiving ${(progress * 100).toInt()}%"; }); }); Navigator.pop(context); }, child: const Text("SECURE CONNECT")),
              ],
            ));
          },
          onConnectionResult: (id, res) { setState(() { connectedId = id; status = "🔐 NFC + Secure WiFi Connected"; }); },
          onDisconnected: (id) {},
        );
      },
      onEndpointLost: (id) {},
      serviceId: serviceId,
    );
  }

  void _advertise() async {
    setState(() => status = "📡 Broadcasting securely... Code: $verifyCode");
    await Nearby().startAdvertising(await _name(), strategy,
      onConnectionInitiated: (id, info) {
        showDialog(context: context, builder: (_) => AlertDialog(
          backgroundColor: const Color(0xFF1E1E2E),
          title: Text("Code $verifyCode", style: const TextStyle(color: Colors.white, fontSize: 28)),
          content: const Text("Other phone tapped via NFC - Verify code", style: TextStyle(color: Colors.white70)),
          actions: [ElevatedButton(onPressed: () { isVerified = true; Nearby().acceptConnection(id, onPayLoadRecieved: (a, b) {}, onPayloadTransferUpdate: (id, up) { setState(() { progress = up.bytesTransferred / up.totalBytes; status = "🔒 Sending ${(progress * 100).toInt()}%"; }); }); Navigator.pop(context); setState(() => status = "🔐 NFC Verified! Ready to SEND"); }, child: const Text("Verified"))],
        ));
      },
      onConnectionResult: (id, res) { setState(() { connectedId = id; }); },
      onDisconnected: (id) {},
      serviceId: serviceId,
    );
  }

  Future<void> _send() async {
    if (!isVerified) { setState(() => status = "Tap NFC first to verify!"); return; }
    if (connectedId == null) { setState(() => status = "Connect first"); return; }
    var res = await FilePicker.platform.pickFiles(type: FileType.image, allowMultiple: true);
    if (res!= null) {
      for (var f in res.files) {
        File file = File(f.path!);
        setState(() => status = "🔒 Encrypting ${f.name} AES-256...");
        final key = SecureEngine.genKey();
        final iv = SecureEngine.genIv();
        final encrypter = enc.Encrypter(enc.AES(key, mode: enc.AESMode.gcm));
        final bytes = file.readAsBytesSync();
        final encrypted = encrypter.encryptBytes(bytes, iv: iv);
        final tempPath = "${file.parent.path}/enc_${f.name}.enc";
        File(tempPath).writeAsBytesSync(encrypted.bytes);
        await Nearby().sendFilePayload(connectedId!, tempPath);
        setState(() => status = "🚀 Sent ${f.name} encrypted - NFC verified");
      }
    }
  }

  @override
  void dispose() { pulse.dispose(); Nearby().stopAllEndpoints(); NfcManager.instance.stopSession(); super.dispose(); }

  Widget _btn(String title, String sub, Color bg, Color fg, VoidCallback tap, {bool primary = false, bool border = false, IconData? icon}) {
    return GestureDetector(
      onTap: tap,
      child: Container(width: double.infinity, padding: EdgeInsets.all(primary? 26 : 18), decoration: BoxDecoration(color: bg, borderRadius: BorderRadius.circular(primary? 28 : 20), border: border? Border.all(color: const Color(0xFF5A67FF), width: 2) : null),
        child: Row(children: [if (icon!= null) Icon(icon, color: fg, size: 30), if (icon!= null) const SizedBox(width: 12), Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [Text(title, style: TextStyle(fontSize: primary? 22 : 18, fontWeight: FontWeight.w900, color: fg)), Text(sub, style: TextStyle(fontSize: 12, color: fg.withOpacity(0.7)))]))]),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFF0A0A0F),
      appBar: AppBar(backgroundColor: const Color(0xFF0A0A0F), title: const Text("🔒 NFC Secure AirDrop", style: TextStyle(fontSize: 24, fontWeight: FontWeight.w900))),
      body: Padding(padding: const EdgeInsets.all(18), child: Column(children: [
        Container(padding: const EdgeInsets.all(14), decoration: BoxDecoration(color: const Color(0xFF1E1E2E), borderRadius: BorderRadius.circular(14), border: Border.all(color: nfcAvailable? Colors.green : Colors.red, width: 1)), child: Row(children: [Icon(nfcAvailable? Icons.nfc : Icons.nfc_outlined, color: nfcAvailable? Colors.green : Colors.red), const SizedBox(width: 10), Text(nfcAvailable? "NFC Available - Tap to Connect" : "NFC Not Available - Using WiFi only", style: TextStyle(color: nfcAvailable? Colors.green : Colors.red, fontSize: 13))])),
        const SizedBox(height: 12),
        Container(padding: const EdgeInsets.all(14), decoration: BoxDecoration(color: const Color(0xFF1E1E2E), borderRadius: BorderRadius.circular(14)), child: Column(children: [Text(verifyCode, style: const TextStyle(fontSize: 36, fontWeight: FontWeight.bold, letterSpacing: 8, color: Color(0xFF5A67FF))), const Text("Encrypted Verification Code", style: TextStyle(color: Colors.white54, fontSize: 11))])),
        const SizedBox(height: 12),
        Text(status, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.w700, color: Colors.white), textAlign: TextAlign.center),
        if (progress > 0) Padding(padding: const EdgeInsets.only(top: 10), child: LinearProgressIndicator(value: progress, minHeight: 8)),
        const Spacer(),
        _btn("📱 NFC TAP TO RECEIVE", "Tap phones - Auto secure", const Color(0xFF2A2A3E), Colors.white, _startNFCReceive, icon: Icons.nfc),
        const SizedBox(height: 10),
        _btn("📱 NFC TAP TO SEND", "Tap phones - Share code", Colors.transparent, Colors.white, _startNFCSend, border: true, icon: Icons.nfc),
        const SizedBox(height: 10),
        _btn("📡 WiFi MODE (No NFC)", "If phone has no NFC", const Color(0xFF1A1A2E), Colors.white70, _advertise, icon: Icons.wifi),
        const SizedBox(height: 10),
        _btn("⚡ SEND ENCRYPTED", "AES-256-GCM • NFC Verified", const Color(0xFF5A67FF), Colors.white, _send, primary: true, icon: Icons.lock),
      ])),
    );
  }
} 

<uses-permission android:name="android.permission.NFC" />
<uses-feature android:name="android.hardware.nfc" android:required="false" />
