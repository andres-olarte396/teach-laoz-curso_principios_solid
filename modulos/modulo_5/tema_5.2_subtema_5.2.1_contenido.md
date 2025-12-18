# Adapter Pattern e ISP

## Problema

Cliente necesita interfaz específica, pero proveedor tiene interfaz diferente.

## Solución: Adapter

```java
// Interfaz que el cliente necesita
interface MediaPlayer {
    void play(String filename);
}

// Clase existente con interfaz diferente
class AdvancedMediaPlayer {
    void playVlc(String filename) { /* ... */ }
    void playMp4(String filename) { /* ... */ }
}

// ✅ Adapter
class MediaAdapter implements MediaPlayer {
    private AdvancedMediaPlayer advancedPlayer;
    
    public void play(String filename) {
        if (filename.endsWith(".vlc")) {
            advancedPlayer.playVlc(filename);
        } else if (filename.endsWith(".mp4")) {
            advancedPlayer.playMp4(filename);
        }
    }
}

// Cliente usa interfaz simple
class AudioPlayer implements MediaPlayer {
    private MediaAdapter adapter;
    
    public void play(String filename) {
        if (filename.endsWith(".mp3")) {
            // Reproducción directa
        } else {
            adapter.play(filename); // Delega a adapter
        }
    }
}
```

## Adapter para ISP

Adapters permiten exponer solo métodos necesarios:

```java
// Fat interface de librería externa
interface ThirdPartyService {
    void method1();
    void method2();
    void method3();
    // ... 20 métodos más
}

// ✅ Adapter expone solo lo necesario
interface SimpleService {
    void doWork();
}

class ServiceAdapter implements SimpleService {
    private ThirdPartyService service;
    
    public void doWork() {
        service.method1();
        service.method2();
        // Combina métodos según necesidad del cliente
    }
}
```

## Resumen

**Adapter** = Traducir interfaces incompatibles, útil para cumplir ISP con código externo.
