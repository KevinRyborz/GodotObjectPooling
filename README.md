# Object Pooling in Godot


## Motivation
In der Spieleentwicklung, insbesondere bei Spielen mit vielen kurzlebigen Objekten (z. B. Projektile, Partikeleffekte, Gegner), kann das ständige Erzeugen (Instanziieren) und Zerstören dieser Objekte zu Leistungseinbußen führen. Jedes instanzierte Objekt wird im Speicher allokiert, initialisiert, zur Szene hinzugefügt und später aufgrund von Ereignissen wie Kollisionen aus dem Speicher entfernt. Wenn dies häufig in kurzer Zeit geschieht, kann es zu spürbaren Rucklern kommen.
Das Object Pooling Pattern ist ein Entwurfsmuster, das dieses Problem minimiert. Anstatt Objekte bei Bedarf neu zu erstellen und später zu zerstören, wird eine vorgegebene Anzahl von Objekten initial erstellt und in einem "Pool" vorgehalten. Wenn ein Objekt benötigt wird, wird es dem Pool entnommen, aktiviert, verwendet und anschließend nicht zerstört, sondern deaktiviert und wieder dem Pool hinzugefügt. Es kann dann erneut aus dem Pool entnommen und wiederverwendet werden.
Stell dir das wie eine Bibliothek vor: Anstatt jedes Mal ein neues Buch zu kaufen (instanziieren) und es nach dem Lesen wegzuwerfen (freigeben), leihst du dir ein Buch aus der Bibliothek (Pool). Nach dem Lesen gibst du es zurück, sodass es für andere verfügbar ist. Die Bibliothek verwaltet die vorhandenen Bücher und gibt sie bei Bedarf aus.

## Das Problem
Ein klassisches Beispiel sind Projektile in Bullet Hell Shootern. In jedem Frame oder in kurzen Abständen werden neue Projektile instanziiert. Wenn sie den Bildschirm verlassen oder ein Ziel treffen, werden sie freigegeben. Bei einer großen Anzahl von Projektilen kann dieser Prozess die CPU belasten und zu Leistungsproblemen führen.
Ohne Object Pooling könnte ein typischer Anwendungsfall in Godot so aussehen:

1. Ein Bedarf entsteht (z. B. der Spieler schießt).
2. Ein neues Objekt (z. B. ein Projektil) wird instanziiert `(PackedScene.instance())`.
3. Das Objekt wird initialisiert und zum Szenenbaum hinzugefügt `(add_child())`.
4. Das Objekt führt seine Logik aus (bewegt sich, prüft Kollisionen).
5. Das Objekt hat seinen Zweck erfüllt oder ist nicht mehr relevant (trifft etwas, verlässt den Bildschirm).
6. Das Objekt wird zur Freigabe vorgemerkt `(queue_free())`.
7. Später im Frame (oder danach) wird der Speicher des Objekts freigegeben.

Dieser ständige Zyklus des Erzeugens und Zerstörens kann erhebliche Verarbeitungszeit beanspruchen und zu Spitzen in der Frametime führen, wenn viele Objekte gleichzeitig erstellt oder freigegeben werden.
Obwohl Godot's Speicherverwaltung, insbesondere in GDScript, oft effizienter ist als in Umgebungen mit traditioneller Garbage Collection, und `queue_free()` asynchron arbeitet, kann der Overhead des Instanziierens von Szenen immer noch relevant sein, wenn es exzessiv betrieben wird.

## Wann sollte Object Pooling verwendet werden?
Object Pooling ist sinnvoll, wenn:

- Häufiges Erzeugen und Zerstören von Objekten stattfindet.
- Hohe Kosten für die Objekterzeugung durch Ressourcenallokierung, Verbindungsaufbau oder komplexe Initialisierung entstehen.
- Begrenzte Anzahl gleichzeitig benötigter Objekte existiert, und die maximale Anzahl eingeschränkt werden kann.
- Vorhersehbare Objektlast vorliegt, sodass die ungefähre Anzahl der benötigten Objekte im Voraus abgeschätzt werden kann.


## Das Pattern
Das Object Pooling Pattern löst dieses Problem, indem es den Lebenszyklus der Objekte verändert:

1. Initialisierung: Zu Beginn eines Levels oder Spielabschnitts wird eine bestimmte Anzahl von Objekten des gewünschten Typs erstellt und in einem Pool gespeichert. Diese Objekte sind zunächst inaktiv, unsichtbar und befinden sich in einem deaktivierten Zustand.
2. Anforderung: Wenn ein Objekt benötigt wird, fragt der anfordernde Code (der "Client", z. B. der Spieler-Charakter, der ein Projektil abfeuern möchte) den Pool nach einem verfügbaren Objekt.
3. Entnahme: Der Pool liefert ein ungenutztes Objekt aus seiner Sammlung.
4. Nutzung: Der Client initialisiert das erhaltene Objekt für seine spezifische Verwendung (z. B. setzt Position, Richtung, Geschwindigkeit) und aktiviert es (macht es sichtbar).
5. Rückgabe: Wenn das Objekt nicht mehr benötigt wird (z. B. das Projektil trifft ein Ziel oder verlässt den Bildschirm), ruft der Client eine Methode am Pool auf, um das Objekt zurückzugeben.
6. Wiederverwertung: Der Pool setzt das zurückgegebene Objekt in seinen ungenutzten Zustand zurück (macht es unsichtbar, deaktiviert es) und fügt es wieder seiner Sammlung hinzu, bereit für die nächste Anforderung.

Der Kern des Patterns ist die Vermeidung des wiederholten Instanziierens und Freigebens zur Laufzeit. Stattdessen wird der Hauptaufwand auf die Initialisierung des Pools verlagert. Während des Spiels werden Objekte lediglich "ausgeliehen" und "zurückgegeben", was in der Regel deutlich performanter ist als das vollständige Erstellen und Zerstören von Knoten im Szenenbaum.

## Implementierung in Godot
Die Implementierung eines Object Pools in Godot kann auf verschiedene Arten erfolgen, oft unter Verwendung von Arrays oder Dictionaries, um die Objekte zu speichern. Für dieses Beispiel konzentrieren wir uns auf zwei Skripte: projectile.gd für die Funktionalität des Projektils und turret.gd, der den Pool verwaltet und Projektile abfeuert.
Object Pool Skript
Nachfolgend ein Beispiel für ein GDScript zur Verwaltung eines Object Pools:
### projectile_pool.gd
    extends Node
    
    var pooled_objects = []
    var object_scene = preload("res://scenes/Projectile.tscn") # Szene des zu poolenden Objekts
    var pool_size = 20 # Startgröße des Pools
    
    func _ready():
        # Pool initial füllen
        for i in range(pool_size):
            var new_object = object_scene.instance()
            add_child(new_object) # Füge Objekt zum Szenenbaum hinzu (kann auch woanders sein)
            new_object.visible = false
            new_object.set_process(false)
            new_object.set_physics_process(false)
            # Optional: Deaktiviere Kollisionen, etc.
            if new_object is CollisionObject2D:
                new_object.set_deferred("monitorable", false)
                new_object.set_deferred("monitoring", false)
            pooled_objects.append(new_object)

## Methode zum Anfordern eines Objekts aus dem Pool
    func get_object():
        for obj in pooled_objects:
            if !obj.visible: # Einfacher Weg zu prüfen, ob das Objekt ungenutzt ist
                obj.visible = true
                obj.set_process(true)
                obj.set_physics_process(true)
                # Optional: Aktiviere Kollisionen, etc.
                if obj is CollisionObject2D:
                    obj.set_deferred("monitorable", true)
                    obj.set_deferred("monitoring", true)
                return obj
    
    # Optional: Wenn Pool leer ist, neues Objekt erstellen (dynamisches Pooling)
    var new_object = object_scene.instance()
    add_child(new_object)
    pooled_objects.append(new_object)
    print("Poolgröße erhöht auf ", pooled_objects.size()) # Zur Info
    return new_object

    

## Methode zum Zurückgeben eines Objekts an den Pool
    func return_object(obj):
        obj.visible = false
        obj.set_process(false)
        obj.set_physics_process(false)
        # Optional: Deaktiviere Kollisionen, etc.
        if obj is CollisionObject2D:
            obj.set_deferred("monitorable", false)
            obj.set_deferred("monitoring", false)
        # Setze das Objekt in einen Standardzustand zurück (Position, Geschwindigkeit, etc.)
        # Dies muss im zurückgegebenen Objekt selbst oder hier geschehen.

## Erklärung

Initialisierung: Der Pool wird in _ready() mit pool_size Instanzen der Projektil-Szene gefüllt. Objekte werden unsichtbar gemacht und deaktiviert, um die Ressourcennutzung zu minimieren.
Objekte anfordern: Die Methode get_object() prüft, ob ein ungenutztes Objekt (basierend auf Sichtbarkeit) verfügbar ist. Falls nicht, wird dynamisch ein neues erstellt.
Objekte zurückgeben: Die Methode return_object() deaktiviert das Objekt und setzt es in einen ungenutzten Zustand zurück, sodass es für zukünftige Anforderungen verfügbar ist.

Diese Implementierung ist einfach, aber effektiv für die meisten Anwendungsfälle in Godot und bietet eine gute Balance zwischen Leistung und Benutzerfreundlichkeit.

Dieser Leitfaden soll Godot-Entwicklern helfen, ihre Spiele mit dem Object Pooling Pattern zu optimieren. Weitere Details findest du in der offiziellen Godot-Dokumentation oder in Community-Ressourcen.
