# FriendlyChat

<!-- Replace X and Title -->

| | |
|---|---|
| **Course** | Mobile Computation |
| **Student(s)** | Tomás Santos |
| **Date** | 2026-05-17 |
| **Repository URL** | github.com/tomassantos2025/build-android-start |

---

## 1. Introduction

This assignment is based on the official **Google Firebase Codelab for Android** — *FriendlyChat* — a hands-on introduction to integrating Firebase services into an Android application. The goal is to build a real-time group chat application that demonstrates core Firebase capabilities including Authentication, Realtime Database, and Cloud Storage.

**Problem description:** Modern mobile applications require reliable, scalable backends for real-time communication. This assignment explores how Firebase provides a complete backend-as-a-service (BaaS) solution, removing the need to manage servers while still supporting real-time data synchronisation across multiple devices.

**Objectives:**
- Integrate Firebase Authentication with email/password and Google Sign-In via FirebaseUI.
- Use Firebase Realtime Database to send and receive chat messages in real time.
- Upload images to Firebase Cloud Storage and reference them in messages.
- Use FirebaseUI's `FirebaseRecyclerAdapter` for reactive, lifecycle-aware list updates.
- Apply View Binding for safe and concise UI access in Android.

---

## 2. System Overview

**FriendlyChat** is a real-time group chat Android application where authenticated users can exchange text messages and images in a shared chat room. All messages are persisted and synchronised via Firebase Realtime Database.

**Main features:**
- **Authentication** — sign in with email/password or Google account via FirebaseUI's pre-built sign-in flow.
- **Real-time messaging** — messages sent by any user appear instantly on all connected clients, powered by Firebase Realtime Database.
- **Image sharing** — users can pick an image from the device gallery; it is uploaded to Firebase Cloud Storage and a placeholder is shown while the upload is in progress.
- **Differentiated message bubbles** — the current user's messages are displayed in blue; others appear in grey.
- **Auto-scroll** — the chat list automatically scrolls to the latest message when new content arrives.
- **Sign out** — accessible from the overflow menu in the action bar.

---

## 3. Architecture and Design

### Architecture

The app follows a lightweight **MVC** structure using Android's Activity model combined with Firebase's reactive data layer:

- **Model** — `FriendlyMessage.kt` (Kotlin data class), directly mapped to/from Firebase Realtime Database nodes.
- **View** — XML layouts with **View Binding** for type-safe access.
- **Controller** — `MainActivity.kt` and `SignInActivity.kt` orchestrate Firebase interactions and UI updates.

The `FirebaseRecyclerAdapter` from FirebaseUI acts as a bridge between the Realtime Database query and the `RecyclerView`, automatically managing data synchronisation and lifecycle.

### Folder Structure

```
app/src/main/
├── AndroidManifest.xml
├── java/com/google/firebase/codelab/friendlychat/
│   ├── MainActivity.kt                  # Chat screen: messages, send, image upload
│   ├── SignInActivity.kt                # FirebaseUI sign-in flow
│   ├── FriendlyMessageAdapter.kt        # FirebaseRecyclerAdapter for chat messages
│   ├── MyButtonObserver.kt              # TextWatcher: enable/disable send button
│   ├── MyOpenDocumentContract.kt        # ActivityResultContract for image picker
│   ├── MyScrollToBottomObserver.kt      # AdapterDataObserver: auto-scroll on new msg
│   └── model/
│       └── FriendlyMessage.kt           # Data model: text, name, photoUrl, imageUrl
└── res/
    ├── layout/
    │   ├── activity_main.xml            # Chat screen layout
    │   ├── activity_sign_in.xml         # Sign-in screen layout
    │   ├── message.xml                  # Text message item layout
    │   └── image_message.xml            # Image message item layout
    ├── drawable/                        # Icons, message bubble backgrounds
    ├── menu/main_menu.xml               # Action bar overflow menu (Sign Out)
    └── values/                          # Strings, colors, styles, dimensions
```

### Key Design Decisions

- **FirebaseUI Auth** — instead of building a custom sign-in UI, FirebaseUI provides a complete, production-grade sign-in flow with email and Google as providers, saving significant development time.
- **FirebaseRecyclerAdapter + `setLifecycleOwner`** — binding the adapter to the Activity lifecycle ensures the Firebase listener starts and stops automatically with `onStart`/`onStop`, preventing memory leaks and unnecessary data usage.
- **Dual view types** — `FriendlyMessageAdapter` uses two `ViewHolder` types (`MessageViewHolder` for text, `ImageMessageViewHolder` for images), allowing a single adapter to handle mixed message content.
- **Temporary placeholder on image upload** — a loading GIF is written to the database immediately when an image is selected, giving the user instant visual feedback before the actual upload completes.
- **`gs://` URI resolution** — the adapter checks if an image URL starts with `gs://` (Cloud Storage URI) and resolves it to an HTTPS download URL before loading with Glide, supporting both storage-hosted and externally hosted images.
- **View Binding** — eliminates `findViewById` calls and null pointer risks across all layouts.

---

## 4. Implementation

### Main Modules

**`SignInActivity`** — on `onStart()`, checks if a user is already authenticated. If not, launches the FirebaseUI sign-in intent with email and Google providers. On a successful sign-in result it navigates to `MainActivity`.

**`MainActivity`** — the core screen. On creation it verifies authentication, initialises the Realtime Database reference (`messages/`), builds a `FirebaseRecyclerOptions` query ordered by insertion, and wires the `FriendlyMessageAdapter` to the `RecyclerView`. The send button pushes a `FriendlyMessage` node to the database. The image button opens the system document picker via `MyOpenDocumentContract`.

**`FriendlyMessageAdapter`** — extends `FirebaseRecyclerAdapter<FriendlyMessage, ViewHolder>`. Overrides `getItemViewType` to distinguish text vs image messages and inflates the appropriate layout. Handles avatar loading with Glide (with circular crop) and sets message bubble colour based on whether the message belongs to the current user.

**`MyButtonObserver`** — a `TextWatcher` that enables/disables the send button and swaps its icon between an active blue arrow and a grey arrow depending on whether the input field has content.

**`MyScrollToBottomObserver`** — an `AdapterDataObserver` that scrolls the `RecyclerView` to the newly inserted position when the user is at the bottom of the list or when the list is first loading.

**`MyOpenDocumentContract`** — extends `ActivityResultContracts.OpenDocument()` adding `Intent.CATEGORY_OPENABLE` to restrict the picker to files that can be opened as a stream.

**`FriendlyMessage`** — a Kotlin data class with default-value parameters (`null`) to satisfy Firebase's requirement for a no-argument constructor during deserialisation.

### Relevant Code Excerpts

```kotlin
// MainActivity.kt — real-time query wired to adapter
val options = FirebaseRecyclerOptions.Builder<FriendlyMessage>()
    .setQuery(messagesRef, FriendlyMessage::class.java)
    .setLifecycleOwner(this)   // auto start/stop with Activity lifecycle
    .build()
```

```kotlin
// MainActivity.kt — two-step image upload (placeholder → real URL)
val tempMessage = FriendlyMessage(null, getUserName(), getPhotoUrl(), LOADING_IMAGE_URL)
db.reference.child(MESSAGES_CHILD).push().setValue(tempMessage, { _, ref ->
    val storageReference = Firebase.storage.getReference(user.uid)
        .child(ref.key!!).child(uri.lastPathSegment!!)
    putImageInStorage(storageReference, uri, ref.key)
})
```

```kotlin
// FriendlyMessageAdapter.kt — resolve gs:// URIs before loading with Glide
if (url.startsWith("gs://")) {
    Firebase.storage.getReferenceFromUrl(url).downloadUrl
        .addOnSuccessListener { uri -> loadWithGlide(view, uri.toString(), isCircular) }
} else {
    loadWithGlide(view, url, isCircular)
}
```

---

## 5. Testing and Validation

### Testing Strategy

- **Manual end-to-end testing** on an Android device/emulator using the debug build.
- **Instrumented test scaffold** — `MainActivityEspressoTest.java` provides a JUnit4 + Espresso test harness targeting `MainActivity`. The test class is structured and ready for test cases to be added; no specific test cases are implemented in this codelab baseline.

### Manual Test Scenarios

| Scenario | Expected Result | Status |
|---|---|---|
| Open app without being signed in | Redirected to sign-in screen | ✅ |
| Sign in with email/password | Navigated to chat screen | ✅ |
| Sign in with Google | Navigated to chat screen | ✅ |
| Send a text message | Message appears instantly in the list | ✅ |
| Other device sends a message | Message appears in real time | ✅ |
| Current user's messages displayed in blue | Blue bubble; others grey | ✅ |
| Send button disabled when input is empty | Grey icon, not clickable | ✅ |
| Pick an image from gallery | Placeholder appears, then real image loads | ✅ |
| Sign out from menu | Redirected to sign-in screen | ✅ |
| Reopen app while still authenticated | Goes directly to chat screen | ✅ |

### Known Limitations

- **No individual chat rooms** — all users share a single global `messages/` node; there is no per-user or per-room isolation.
- **No message deletion or editing** — once sent, messages are permanent.
- **No pagination** — the entire message history is loaded on connect; for large datasets this could become slow.
- **Instrumented tests are empty** — the Espresso test class exists but contains no assertions.
- **Single initial commit** — does not reflect incremental development progress.

---

## 6. Usage Instructions

### Requirements

- Android Studio (Hedgehog or later recommended)
- Android SDK 36 (`compileSdk`), minimum SDK 23 (Android 6.0)
- A Firebase project with the following services enabled:
  - **Authentication** — Email/Password and Google providers
  - **Realtime Database** — with rules allowing authenticated reads/writes
  - **Cloud Storage** — with rules allowing authenticated uploads
- `google-services.json` placed inside the `app/` directory

### Setup

1. Clone or download the repository and open it in Android Studio.
2. In the [Firebase Console](https://console.firebase.google.com), create a project (or use an existing one).
3. Register an Android app with package name `com.google.firebase.codelab.friendlychat`, download `google-services.json`, and place it in `app/`.
4. Enable **Email/Password** and **Google** sign-in methods under Authentication → Sign-in method.
5. Set Realtime Database rules:
   ```json
   {
     "rules": {
       ".read": "auth != null",
       ".write": "auth != null"
     }
   }
   ```
6. Set Cloud Storage rules:
   ```
   rules_version = '2';
   service firebase.storage {
     match /b/{bucket}/o {
       match /{allPaths=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
7. Sync Gradle (`File > Sync Project with Gradle Files`) and build the project.
8. Run on an emulator or physical device (API 23+).

---

## 7. Prompting Strategy

AI tools (Claude) were used to assist with specific implementation details during development. Prompts evolved from broad questions to focused, context-specific requests:

**Example prompts used:**

- *"How does FirebaseRecyclerAdapter work with setLifecycleOwner — does it handle onStart/onStop automatically?"* — confirmed that binding the lifecycle owner removes the need for manual `startListening()`/`stopListening()` calls.
- *"Why does my Kotlin data class fail to deserialise from Firebase Realtime Database?"* — identified the need for default parameter values (`= null`) to satisfy the no-arg constructor requirement.
- *"How do I resolve a `gs://` Firebase Storage URL to an HTTPS download URL in Android?"* — produced the `getReferenceFromUrl().downloadUrl` pattern used in the adapter.
- *"What is the correct way to use ActivityResultContracts.OpenDocument to restrict to openable files?"* — led to the `MyOpenDocumentContract` implementation with `CATEGORY_OPENABLE`.
- *"Write a README for a FriendlyChat Firebase codelab Android project following this template: [template pasted]"* — used for this documentation.

---

## 8. Autonomous Agent Workflow

AI contributed across the following development stages:

- **Planning** — helped map Firebase services (Auth, Realtime Database, Storage) to application features before implementation.
- **Coding** — generated boilerplate for the `FirebaseRecyclerOptions` setup, the two-step image upload pattern (placeholder → real URL), and the `MyScrollToBottomObserver` logic.
- **Debugging** — identified the root cause of Firebase deserialisation failures (missing default constructor) and the `stopListening()` crash (missing `notifyItemRangeRemoved`).
- **Documentation** — assisted in writing this README following the provided template.

All AI-generated code was reviewed, tested, and integrated manually.

---

## 9. Verification of AI-Generated Artifacts

AI-generated code and guidance was verified through:

- **Cross-referencing with official Firebase documentation** and the original Google Codelab instructions to confirm correctness of API usage.
- **Runtime testing** on a device, exercising each feature listed in Section 5.
- **Android Studio lint and compiler checks** — all warnings and errors were resolved before considering a feature complete.
- **Firebase Console inspection** — the Realtime Database and Storage dashboards were used to confirm data was being written and structured correctly (e.g., per-user storage paths, message nodes).

---

## 10. Human vs AI Contribution

| Component | Primary Contributor |
|---|---|
| Overall app architecture and Firebase service selection | Human |
| `SignInActivity` — FirebaseUI sign-in flow | Human |
| `MainActivity` — lifecycle, database query, send message | Human |
| `FriendlyMessageAdapter` — dual view types, avatar loading | Human + AI-assisted |
| `gs://` URI resolution pattern in adapter | AI-assisted |
| `MyButtonObserver` and `MyScrollToBottomObserver` | Human |
| Two-step image upload (placeholder → Cloud Storage) | AI-assisted |
| `FriendlyMessage` data class (default params for Firebase) | AI-assisted |
| XML layouts and drawables | Human |
| This README | AI-assisted (Claude) |

---

## 11. Ethical and Responsible Use

- All AI suggestions were reviewed and understood before integration; no code was committed blindly.
- AI occasionally suggested deprecated APIs (e.g., `startActivityForResult`) — these were identified via Android documentation and updated to `ActivityResultContracts` equivalents.
- No sensitive data (API keys, `google-services.json`, Firebase credentials) was shared with AI tools.
- The single-room chat design (shared global `messages/` path) was highlighted by AI as a privacy concern in production — acknowledged as an acceptable trade-off within the codelab scope.
- All final code, including AI-assisted sections, is the responsibility of the student.

---

## 12. Version Control and Commit History

The project does not include a `.git` history in the submitted archive. Version control with Git should be used throughout development with descriptive, incremental commits such as:

- `feat: add FirebaseUI sign-in with email and Google providers`
- `feat: integrate FirebaseRecyclerAdapter for real-time messages`
- `feat: implement image upload to Cloud Storage`
- `fix: resolve gs:// URI before loading with Glide`
- `fix: notify RecyclerView on stopListening to prevent crash`

> **Note:** Future submissions should include a full commit history reflecting continuous, incremental work rather than a single final commit.

---

## 13. Difficulties and Lessons Learned

- **Firebase deserialisation** — learned that Kotlin data classes require default values (`= null`) for all parameters to allow Firebase to instantiate them via reflection. Without defaults, the SDK throws a runtime exception.
- **FirebaseRecyclerAdapter lifecycle** — understanding when to call `startListening()`/`stopListening()` vs using `setLifecycleOwner()` was initially confusing. The lifecycle owner approach is cleaner and less error-prone.
- **`stopListening()` crash** — overriding `stopListening()` to call `notifyItemRangeRemoved` after `super.stopListening()` was necessary to prevent a `RecyclerView` inconsistency crash when the adapter cleared its snapshots without notifying the view.
- **`gs://` vs HTTPS URLs** — Cloud Storage download URLs returned from the SDK are `gs://` scheme URIs, which Glide cannot load directly. They must first be resolved to HTTPS download URLs.
- **Two-step image upload UX** — writing a placeholder message to the database immediately on image selection (before the upload completes) provides a much better user experience than waiting for the full upload before showing anything.

---

## 14. Future Improvements

- **Multiple chat rooms** — add a room/channel selector so users can create or join named conversations.
- **Per-user message isolation** — restrict read access so users only see messages in rooms they have joined.
- **Message deletion** — allow users to delete their own messages from the Realtime Database.
- **Pagination / lazy loading** — use `limitToLast(N)` and load more messages as the user scrolls up, rather than loading the entire history.
- **Push notifications** — integrate Firebase Cloud Messaging (FCM) to notify users of new messages when the app is in the background.
- **Message timestamps** — store and display a server timestamp (`ServerValue.TIMESTAMP`) with each message.
- **Typing indicators** — use a dedicated Realtime Database node to show when another user is composing a message.
- **Comprehensive Espresso tests** — fill out the `MainActivityEspressoTest` scaffold with real assertions covering sign-in flow, message sending, and UI state.

---

## 15. AI Usage Disclosure (Mandatory)

| Tool | How it was used |
|---|---|
| **Claude (Anthropic)** | Code assistance (Firebase deserialisation, image upload pattern, `gs://` URI resolution), debugging guidance, and README generation |

All AI-generated content was reviewed, adapted, and tested by the student before inclusion in the project. The student takes full responsibility for all code, design decisions, and documentation. AI tools were used as a productivity aid, not as a replacement for understanding.
