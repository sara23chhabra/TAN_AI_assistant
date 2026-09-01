# TAN AI Assistant

TAN is a Flutter-based AI shopping assistant prototype designed for voice-first product discovery and conversational product interaction.

The current prototype combines:

- **Flutter / Dart** for the mobile application
- **Speech-to-Text (STT)** for voice input
- **Qwen 3 (1.7B)** running locally through **Ollama** for intent/query understanding
- **DummyJSON Products API** for product search and product data
- **Flutter TTS** for spoken responses
- A modular **Vision Service** for analyzing product images
- Conversational product-reference handling such as "the first one", "the second one", or referring back to a previously selected product

> **Project status:** This is an active proof-of-concept (POC). Voice input and product search are working on a physical iPhone. The local Qwen/Ollama connection must be reachable from the phone over the same Wi-Fi network. The Vision integration is still being debugged on iOS.

---

## Architecture

```text
                    ┌─────────────────────┐
                    │     User Voice      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Flutter STT Layer  │
                    │ speech_to_text     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     main.dart      │
                    │ Conversation/UI    │
                    └──────┬───────┬──────┘
                           │       │
              ┌────────────┘       └─────────────┐
              ▼                                  ▼
     ┌──────────────────┐               ┌──────────────────┐
     │ Qwen via Ollama  │               │ Product Service  │
     │ Intent + Query   │               │ DummyJSON API    │
     └────────┬─────────┘               └────────┬─────────┘
              │                                  │
              └──────────────┬───────────────────┘
                             ▼
                    ┌─────────────────────┐
                    │ Product Results     │
                    │ + Reference Memory  │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
             ┌─────────────┐      ┌─────────────┐
             │ Vision      │      │ Flutter TTS │
             │ Service     │      │ Spoken      │
             │             │      │ Response    │
             └─────────────┘      └─────────────┘
```

---

## Project Structure

The application is intentionally split into modular services rather than keeping all logic inside `main.dart`.

Typical responsibilities are:

### `lib/main.dart`

The main Flutter application layer.

It handles:

- App UI
- Voice interaction
- Speech-to-text lifecycle
- Sending the user's request into the product/search pipeline
- Displaying product results
- Product selection/reference handling
- Text-to-speech output
- Connecting the different services together

### Product service

Responsible for communicating with DummyJSON and turning the API response into Dart product maps.

Conceptually:

```text
User query
    ↓
Product API request
    ↓
JSON response
    ↓
jsonDecode()
    ↓
List<Map<String, dynamic>>
    ↓
Flutter product UI
```

### Vision service

A separate module for product-image analysis.

It receives an image URL and communicates with the iOS vision layer through a Flutter `MethodChannel`.

The iOS side registers the channel as:

```text
tan/vision
```

The vision layer is intended to return information describing/analyzing the product image.

**Current status:** the integration is present, but iOS has returned:

```text
VISION_FAILED
Failed to create espresso context.
```

This is being treated as an iOS/native Vision integration issue rather than a DummyJSON product-search issue.

### Speech service / STT

Uses the phone's speech-recognition capabilities to convert spoken input into text.

Example:

```text
User:
"Hello can you show me some laptops"

STT:
"Hello can you show me some laptops"
```

### TTS

Uses `flutter_tts` to speak responses back to the user.

---

## AI Query Pipeline

When the user says something such as:

> "Hey, I want a mascara"

the application follows this flow:

```text
Voice input
   ↓
Speech-to-text
   ↓
Qwen
   ↓
Intent + product extraction
   ↓
"mascara"
   ↓
DummyJSON product search
   ↓
Product results
   ↓
Display + optional spoken response
```

Qwen returns structured information such as:

```json
{
  "new_product": true,
  "intent": "product_search",
  "product": "mascara",
  "brand": null,
  "color": null,
  "maximum_price": null,
  "platform": null
}
```

The product field is then used to construct the product-search request.

---

## Local Qwen / Ollama Setup

Qwen is currently running locally rather than being bundled into the iPhone application.

The model currently used is:

```text
qwen3:1.7b
```

Ollama normally exposes its API at:

```text
http://localhost:11434
```

### Important for a physical iPhone

`localhost` on the iPhone means **the iPhone itself**, not the Mac.

Therefore this does **not** work when the phone is calling a Qwen server running on the Mac:

```text
http://localhost:11434/api/chat
```

Instead, the Mac must expose Ollama on the local network and the Flutter app must use the Mac's LAN IP:

```text
http://<MAC_IP>:11434/api/chat
```

For example:

```text
http://192.168.x.x:11434/api/chat
```

The Mac and iPhone must be connected to the same Wi-Fi network.

### Test Ollama on the Mac

```bash
curl http://localhost:11434/api/tags
```

The response should contain:

```text
qwen3:1.7b
```

### Test Ollama over the LAN

From a device that can reach the Mac:

```bash
curl http://<MAC_IP>:11434/api/tags
```

If this succeeds, the phone should be able to reach the Ollama API as well, subject to local network/firewall settings.

---

## Why `localhost` caused the physical-phone error

On the physical iPhone we initially saw:

```text
ClientException with SocketException:
Connection refused

uri=http://localhost:11434/api/chat
```

This happened because the application was trying to find Ollama on the iPhone.

Ollama was actually running on the Mac.

The solution is to expose Ollama to the local network and point Flutter at the Mac's LAN IP.

---

## Product Data

The prototype currently uses the DummyJSON Products API.

Example request:

```text
https://dummyjson.com/products/search?q=mascara&limit=30
```

The API returns product information including fields such as:

- Product ID
- Product name
- Price
- Rating
- Product image
- Category
- Description

The application converts the JSON response into Dart maps before displaying the products.

---

## Product References

TAN supports conversational references to products already shown to the user.

Examples:

```text
"the first one"
"the second one"
"the third one"
"tell me more about that one"
```

The reference resolver maps these requests to products in the currently displayed product list.

The current implementation supports positional references through a defined range (including first through twentieth).

This is **not a hard limit on the number of products the API can return**. It is a limit of the currently implemented positional-reference vocabulary.

The current reference system is primarily deterministic/pattern-based. More advanced semantic references such as:

```text
"the cheaper Mac"
"the one with the highest rating"
"the laptop you mentioned earlier"
```

would require additional contextual reasoning.

---

## Voice Interaction

The app uses speech recognition to provide hands-free interaction.

A successful run produces logs similar to:

```text
STT AVAILABLE: true
STT STATUS: available
STT STATUS: listening
STT RESULT: Hello can you show me some laptops
```

The final recognized text is passed into the TAN search pipeline.

---

## iOS Development

The project is currently being tested on physical iPhones as well as the iOS Simulator.

For a physical device:

1. Connect the iPhone to the Mac.
2. Trust the Mac on the iPhone if prompted.
3. Enable Developer Mode when required.
4. Ensure Xcode signing is configured.
5. Verify the phone appears in:

```bash
flutter devices
```

6. Run:

```bash
flutter run -d <DEVICE_ID>
```

---

## Installing Dependencies

From the project root:

```bash
flutter clean
flutter pub get
```

For iOS:

```bash
cd ios
pod install
cd ..
```

Then:

```bash
flutter run -d <DEVICE_ID>
```

---

## Known Issues

### 1. Vision integration

The current iOS Vision integration has produced:

```text
VISION_FAILED
Failed to create espresso context.
```

The product-search pipeline itself can still work independently of this failure.

### 2. `flutter_tts` and Swift Package Manager

Flutter currently reports:

```text
The following plugins do not support Swift Package Manager for ios:
- flutter_tts
```

This is a warning in the current setup, but Flutter indicates that this may become an error in a future version.

### 3. Ollama is currently local

The current architecture depends on Ollama running on a Mac.

For a standalone production mobile application, the AI backend should eventually be moved to a remotely accessible server/API or another architecture that does not require the user's phone and development Mac to be on the same Wi-Fi network.

### 4. DummyJSON is prototype data

DummyJSON is being used as a mock product source for development.

A production version should replace it with the intended real product/catalog backend.

---

## Current End-to-End Goal

The intended interaction is:

```text
User speaks
    ↓
STT
    ↓
Qwen understands intent
    ↓
Product API search
    ↓
Products displayed
    ↓
User refers to a product conversationally
    ↓
TAN resolves the reference
    ↓
Product information / image analysis
    ↓
TTS speaks the response
```

The project is currently at the **POC / integration stage**, with the voice → Qwen → product-search pipeline being actively tested on physical iOS devices.

---

## Development Notes

When debugging, Flutter logs are intentionally verbose.

Useful log prefixes include:

```text
TAN USER INPUT
QWEN STATUS
QWEN RAW RESPONSE
TAN PRODUCT
TAN REFERENCE CHECK
TAN VISION SERVICE
STT STATUS
STT RESULT
```

These make it easier to determine which layer is failing:

```text
STT failure
    ↓
speech recognition problem

QWEN connection failure
    ↓
Ollama/network problem

Product API failure
    ↓
product backend problem

Vision failure
    ↓
iOS/native Vision integration problem

TTS failure
    ↓
flutter_tts/iOS speech problem
```

---

## Future Improvements

- Replace local Ollama dependency with a production AI backend
- Improve semantic product-reference resolution
- Add richer image/product understanding
- Resolve the iOS Vision integration issue
- Replace DummyJSON with a real product catalog
- Add product filtering and sorting
- Add persistent conversational memory
- Improve error handling and offline behavior
- Add automated tests
- Prepare production signing and App Store deployment

---

## Team

TAN AI Assistant is being developed collaboratively as a mobile AI shopping-assistant proof of concept.

Repository:

https://github.com/sara23chhabra/TAN_AI_assistant
