# 🔐 Lab 24 — JNIDemo (JNI + Anti-Debug Native)

---

## 📌 Présentation

Ce laboratoire prolonge le projet **JNIDemo** en ajoutant une couche de sécurité native en **C++**.

L’objectif est de détecter :

* la présence d’un débogueur ;
* l’utilisation d’outils d’instrumentation.

💡 Idée clé : déplacer les contrôles sensibles en natif pour les rendre plus difficiles à contourner.

---

## 🎯 Objectifs pédagogiques

À la fin de ce lab, vous serez capable de :

* intégrer un contrôle anti-debug en C++ ;
* comprendre `ptrace` ;
* analyser `/proc/self/maps` ;
* retourner un booléen vers Java ;
* adapter le comportement de l’application ;
* utiliser Logcat pour le debug natif.

---

## ⚙️ Fonctionnalité principale

Ajout d’une méthode JNI :

```java
public native boolean isDebugDetected();
```

Elle vérifie :

* présence d’un debugger
* présence de bibliothèques suspectes

---

## 🧰 Prérequis

* Projet JNIDemo fonctionnel
* Android Studio
* NDK, CMake, LLDB

---

## 🏗️ Architecture

```
MainActivity
   ↓
isDebugDetected()
   ↓
libnative-lib.so
   ↓
C++ checks
   ↓
boolean → Java
   ↓
UI adapte son comportement
```

---

## 🚀 Étape 1 — Réutiliser le projet

Vérifier :

* native-lib.cpp
* CMakeLists.txt
* System.loadLibrary("native-lib")

---

## ⚙️ Étape 2 — CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.22.1)

project("jnidemo")

add_library(
        native-lib
        SHARED
        native-lib.cpp)

find_library(
        log-lib
        log)

target_link_libraries(
        native-lib
        ${log-lib})
```

---

## 🧠 Étape 3 — Logique défensive

### 1. Détection debug

Utilise `ptrace` pour détecter un debugger.

### 2. Inspection mémoire

Lecture de `/proc/self/maps` pour détecter :

* frida
* xposed
* gdb
* magisk

### 3. Réaction

* log
* blocage logique
* affichage UI

---

## 💻 Étape 4 — Code natif complet

```cpp
#include <jni.h>
#include <string>
#include <cstring>
#include <cstdio>
#include <cstdlib>
#include <android/log.h>
#include <sys/ptrace.h>
#include <unistd.h>

#define LOG_TAG "ANTI_DEBUG"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN, LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)

static bool isBeingTraced() {
    long result = ptrace(PTRACE_TRACEME, 0, 0, 0);
    if (result == -1) {
        LOGE("Etat suspect : trace/debug detecte");
        return true;
    }
    LOGI("Aucun trace/debug detecte via ptrace");
    return false;
}

static bool containsSuspiciousLibraryNames() {
    FILE* maps = fopen("/proc/self/maps", "r");
    if (!maps) {
        LOGW("Impossible d'ouvrir /proc/self/maps");
        return false;
    }

    char line[512];

    while (fgets(line, sizeof(line), maps)) {
        if (strstr(line, "frida") ||
            strstr(line, "xposed") ||
            strstr(line, "libfrida") ||
            strstr(line, "gdbserver") ||
            strstr(line, "libgdb") ||
            strstr(line, "magisk")) {
            LOGE("Signature suspecte trouvee : %s", line);
            fclose(maps);
            return true;
        }
    }

    fclose(maps);
    return false;
}

extern "C"
JNIEXPORT jboolean JNICALL
Java_com_example_jnidemo_MainActivity_isDebugDetected(
        JNIEnv* env,
        jobject /* this */) {

    bool traced = isBeingTraced();
    bool suspicious = containsSuspiciousLibraryNames();

    if (traced || suspicious) {
        LOGE("DEBUG detecte");
        return JNI_TRUE;
    }

    return JNI_FALSE;
}

extern "C"
JNIEXPORT jstring JNICALL
Java_com_example_jnidemo_MainActivity_helloFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    return env->NewStringUTF("Hello from JNI");
}

extern "C"
JNIEXPORT jint JNICALL
Java_com_example_jnidemo_MainActivity_factorial(
        JNIEnv* env,
        jobject /* this */,
        jint n) {

    if (n < 0) return -1;

    long long fact = 1;
    for (int i = 1; i <= n; i++) {
        fact *= i;
    }

    return fact;
}
```

---

## ⚠️ Bonnes pratiques

* limiter JNI
* séparer logique native
* journaliser
* éviter crash brutal

---

## ✅ Conclusion

Ce lab montre :

* comment protéger une app Android
* comment utiliser JNI pour sécurité
* comment détecter debug/instrumentation

⚠️ Important : ce n’est pas une sécurité absolue, mais une couche de défense.

---

## 📌 Suite

👉 Envoie-moi les screenshots suivants (MainActivity, UI, logs)
Je vais compléter le README avec :

* code Java complet
* affichage UI
* gestion du résultat anti-debug
