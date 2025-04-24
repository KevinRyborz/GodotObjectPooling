# Object Pooling in Godot

## Motivation

In der Spieleentwicklung führen Spiele mit vielen kurzlebigen Objekten – wie Projektilen, Partikeleffekten oder Gegnern – oft zu Leistungsproblemen, wenn diese Objekte ständig erstellt und zerstört werden. Jeder dieser Vorgänge umfasst mehrere ressourcenintensive Schritte:

- Speicherallokation: Das Objekt wird im Speicher reserviert.
- Initialisierung: Eigenschaften wie Position oder Geschwindigkeit werden gesetzt.
- Szenenintegration: Das Objekt wird in den Szenenbaum eingefügt.
- Freigabe: Nach Ereignissen wie Kollisionen oder Ablauf wird das Objekt aus dem Speicher entfernt.

Häufiges Ausführen dieser Schritte in kurzer Zeit kann zu spürbaren Rucklern oder Verzögerungen führen, da die Prozesse die CPU und den Speicher belasten.
<br>
<br>
## Das Problem
Ein klassisches Beispiel sind Projektile in Bullet Hell Shootern. In jedem Frame oder in kurzen Abständen werden neue Projektile instanziiert. Wenn sie den Bildschirm verlassen oder ein Ziel treffen, werden sie freigegeben. Bei einer großen Anzahl von Projektilen kann dieser Prozess die CPU belasten und zu Leistungsproblemen führen.
Ohne Object Pooling könnte ein typischer Anwendungsfall in Godot so aussehen:

<br>

1. Ein Bedarf entsteht (z. B. der Spieler schießt).
2. Ein neues Objekt (z. B. ein Projektil) wird instanziiert `(PackedScene.instance())`.
3. Das Objekt wird initialisiert und zum Szenenbaum hinzugefügt `(add_child())`.
4. Das Objekt führt seine Logik aus (bewegt sich, prüft Kollisionen).
5. Das Objekt hat seinen Zweck erfüllt oder ist nicht mehr relevant (trifft etwas, verlässt den Bildschirm).
6. Das Objekt wird zur Freigabe vorgemerkt `(queue_free())`.
7. Später im Frame (oder danach) wird der Speicher des Objekts freigegeben.

<br>

Dieser ständige Zyklus des Erzeugens und Zerstörens kann erhebliche Verarbeitungszeit beanspruchen und zu Spitzen in der Frametime führen, wenn viele Objekte gleichzeitig erstellt oder freigegeben werden.

> [!IMPORTANT]
> Obwohl Godot's Speicherverwaltung, insbesondere in GDScript, oft effizienter ist als in Umgebungen mit traditioneller Garbage Collection, und `queue_free()` asynchron arbeitet, kann der Overhead des Instanziierens von Szenen immer noch relevant sein, wenn es exzessiv betrieben wird.
<br>

## Das Pattern

Das **Object Pooling Pattern** ist ein Entwurfsmuster, das diese Leistungseinbußen minimiert. Statt Objekte bei Bedarf neu zu erstellen und anschließend zu zerstören, wird eine feste Anzahl von Objekten zu Spielbeginn vorbereitet und in einem Pool gespeichert. Der Lebenszyklus eines Objekts im Pool läuft wie folgt ab:

1. **Entnahme**: Ein inaktives Objekt wird aus dem Pool entnommen.
2. **Aktivierung**: Das Objekt wird an die benötigte Stelle gesetzt und die notwendigen Komponenten aktiviert.
3. **Nutzung**: Das Objekt führt seine funktion aus (z.B. ein Projektil das sich bewegt)
4. **Rückgabe**: Nach Gebrauch wird das Objekt deaktiviert (unsichtbar und inaktiv) und in den Pool zurückgegeben, bereit für die Wiederverwendung.

Der Kern des Patterns ist die **Vermeidung des wiederholten Instanziierens und Freigebens zur Laufzeit**. Stattdessen wird der Hauptaufwand auf die Initialisierung des Pools verlagert. Während des Spiels werden Objekte lediglich "ausgeliehen" und "zurückgegeben", was in der Regel deutlich performanter ist als das vollständige Erstellen und Zerstören von Knoten im Szenenbaum.

![image](https://github.com/user-attachments/assets/86399fed-82e2-4dfa-b6a7-aa8426538135)
> source: https://metabox.io/object-pool-pattern/

<br>
<br>

## Wann sollte Object Pooling verwendet werden?
Object Pooling kann sinvoll sein wenn:

- [ ] **Häufiges Erzeugen und Zerstören** von Objekten stattfindet, z. B. bei Projektilen in einem Bullet-Hell-Shooter, die schnell gespawnt und nach Kollisionen oder Verlassen des Bildschirms entfernt werden.
- [ ] **Hohe Kosten für die Objekterzeugung** entstehen, etwa durch komplexe Initialisierungen wie das Laden von komplexen Klassen.
- [ ] **Eine begrenzte Anzahl** von Objekten benötigt wird, z. B. bei einer festen Anzahl von Gegnern.
- [ ] **Vermeidung von Speicherfragmentierung**, z. B. in Spielen mit langen Laufzeiten, wie Roguelikes, wo häufiges Allokieren und Freigeben von Speicher die Performance langfristig beeinträchtigen kann.

<br>

## Pseudocode

    // Globale Variablen
    pool = leeres Array        // Speichert alle Objekte im Pool
    pool_size = 10             // Anzahl der Objekte im Pool
    
    // Initialisierung des Pools (wird einmal zu Spielbeginn ausgeführt)
    Funktion initialisiere_pool():
        Für i von 1 bis pool_size:
            objekt = erstelle_neues_objekt()  // z. B. ein Projektil
            objekt.deaktiviere()             // Objekt unsichtbar und inaktiv
            pool.hinzufügen(objekt)          // Objekt zum Pool hinzufügen
    
    // Objekt aus dem Pool holen
    Funktion hole_objekt():
        Für jedes objekt in pool:
            Wenn objekt.ist_inaktiv():
                objekt.aktiviere()           // Objekt sichtbar und aktiv
                Rückgabe objekt              // Gibt das Objekt zurück
        Rückgabe null                        // Kein Objekt verfügbar
    
    // Objekt zurück in den Pool geben
    Funktion gib_objekt_zurück(objekt):
        objekt.deaktiviere()                 // Objekt unsichtbar und inaktiv
        pool.hinzufügen(objekt)              // Objekt wieder im Pool
    
    // Beispiel für ein Objekt (z. B. Projektil)
    Klasse Objekt:
        ist_aktiv = falsch
        
        Funktion aktiviere():
            ist_aktiv = wahr
            sichtbar = wahr
            // Setze Eigenschaften wie Position, Geschwindigkeit
        
        Funktion deaktiviere():
            ist_aktiv = falsch
            sichtbar = falsch
            // Setze Eigenschaften zurück
    
    
    Funktion spieler_schießt():
        projektil = hole_objekt()
        Wenn projektil != null:
            projektil.setze_position(spieler.position)
            projektil.setze_richtung(vorwärts)
            // Projektil fliegt jetzt
    
    Funktion projektil_fertig(projektil): // z. B. nach Kollision oder Zeit
        gib_objekt_zurück(projektil)
<br>

# Implementierung in Godot
Die Implementierung eines Object Pools in Godot kann auf verschiedene Arten erfolgen, oft unter Verwendung von Arrays oder Dictionaries, um die Objekte zu speichern. Für dieses Beispiel konzentrieren wir uns auf zwei Skripte: `projectile.gd` für die Funktionalität des Projektils und `turret.gd`, der den Pool verwaltet und Projektile abfeuert.

### projectile.gd
    
    extends Node3D
    
    var speed = 20.0
    var direction = Vector3.FORWARD 
    var is_active = false 
    
    @onready var collision_shape = $Area3D/CollisionShape3D
    @onready var mesh_instance = $MeshInstance3D  
    @onready var timer = $Timer
    
    func _process(delta):
    	if is_active:
    		global_translate(direction * speed * delta)
    
    func activate(pos: Vector3, dir: Vector3):
    	global_position = pos
    	direction = dir.normalized()
    	is_active = true
    	visible = true
    	timer.start()
    	if collision_shape:
    		collision_shape.disabled = false
    
    func deactivate():
    	is_active = false
    	visible = false
    	if collision_shape:
    		collision_shape.disabled = true
    
    
    func _on_timer_timeout():
    	if is_active:
    		deactivate()
    
    func _on_area_3d_body_entered(body: Node3D) -> void:
    	if is_active:
    		deactivate()

> [!NOTE]
> ### Erklärung zu `projectile.gd`
<details>
<summary>Aufklappen</summary>

    var speed = 20.0
    var direction = Vector3.FORWARD 
    var is_active = false 
Hier werden die Geschwindigkeit, die Schussrichtung und die bool, welche für die Aktivierung der Projektile zuständig ist, gesetzt. 

---
      func _process(delta):
        	if is_active:
        		global_translate(direction * speed * delta)
In der `process` Funktion (jeden tick) wird geprüft ob das Projektil aktiviert ist. Wenn ja soll es in die Schussrichtung mit der zuvor gesetzen Geschwindigkeit fliegen.

---
    func activate(pos: Vector3, dir: Vector3):
    	global_position = pos
    	direction = dir.normalized()
    	is_active = true
    	visible = true
    	timer.start()
    	if collision_shape:
    		collision_shape.disabled = false
    
    func deactivate():
    	is_active = false
    	visible = false
    	if collision_shape:
    		collision_shape.disabled = true
Die Aktivieren und Deaktivieren Funktionen steuern ob die Projektile sichtbar sind und ob mit ihnen kollidiert werden kann. Da die bool `is_active` zu Beginn `false` ist kummert sich die `_process` Funktion darum die Projektile standardmäßig zu deaktivieren. 

---
      func _on_timer_timeout():
        	if is_active:
        		deactivate()
    
    func _on_area_3d_body_entered(body: Node3D) -> void:
    	if is_active:
    		deactivate()
Wenn ein Projektil instanziert wurde gibt es zwei Möglichkeiten wie es deaktiviert werden kann. Wenn der festgelegte Timer abläuft wird es deaktiviert. Das gleiche gilt für den Fall, dass das Projektil mit etwas kollidiert. Auch hier wird die `is_active` bool auf false gesetzt. 

---


</details>

---

### turret.gd
    
    extends Node3D
    
    @export var projectile_scene: PackedScene
    @export var pool_size = 20
    @export var projectiles_per_shot = 5
    @export var spread_angle_degrees = 30.0
    @export var projectile_speed = 20.0
    
    var projectile_pool = []
    
    @onready var spawn_point: Node3D = self
    
    func _ready():
    	if projectile_scene == null:
    		print("Projectile scene not set!")
    		return
    	for i in range(pool_size):
    		var projectile = projectile_scene.instantiate()
    		add_child(projectile)
    		projectile.deactivate()
    		projectile_pool.append(projectile)
    
    func _input(event):
    	if event is InputEventMouseButton and event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
    		spawn_projectiles()
    
    func count_inactive_projectiles():
    	var inactive_count = 0
    	for projectile in projectile_pool:
    		if not projectile.is_active:
    			inactive_count += 1
    	return inactive_count
    
    func spawn_projectiles():
    	if count_inactive_projectiles() < projectiles_per_shot:
    		print("Nicht genügend inaktive Projektile! Benötigt:", projectiles_per_shot)
    		return
    
    	var base_direction = -global_transform.basis.z
    	for i in range(projectiles_per_shot):
    		var projectile = get_pooled_projectile()
    		if projectile:
    			var rotation_transform = Transform3D().rotated(Vector3.UP, 0)
    			var spawn_direction = rotation_transform.basis * base_direction
    			projectile.speed = projectile_speed
    			projectile.activate(spawn_point.global_position, spawn_direction)
    
    func get_pooled_projectile():
    	for projectile in projectile_pool:
    		if not projectile.is_active:
    			return projectile
    	return null
     
> [!NOTE]
> ### Erklärung zu `turret.gd`
<details>
<summary>Aufklappen</summary>

    @export var projectile_scene: PackedScene
    @export var pool_size = 20
    @export var projectiles_per_shot = 5
    @export var spread_angle_degrees = 30.0
    @export var projectile_speed = 20.0
    
    var projectile_pool = []
    
    @onready var spawn_point: Node3D = self
    
    func _ready():
    	if projectile_scene == null:
    		print("Projectile scene not set!")
    		return
    	for i in range(pool_size):
    		var projectile = projectile_scene.instantiate()
    		add_child(projectile)
    		projectile.deactivate()
    		projectile_pool.append(projectile) 
    
Hier werden die Grundvariablen für den Projectile Pool gesetzt. Durch `@export` können diese auch über den Editor eingestellt werden. Zusätzlich wird ein leeres Array `projectile_pool` erstellt welches Referenzen zu allen Projectiles besitzt. 
In der `_ready()` Funktion werden anschließend mittels eines `for` Loops die Projektile instanziert und deaktiviert um Kollisionen zu verhindern und sie unsichtbar zu machen. In jedem Loop durchlauf werden außerdem mit `projectile_pool.append(projectile)` die
instanzierten Projektile zum Array hinzugefügt.

---

    func count_inactive_projectiles():
    	var inactive_count = 0
    	for projectile in projectile_pool:
    		if not projectile.is_active:
    			inactive_count += 1
    	return inactive_count

Die `count_inactive_projectiles` Funktion wird verwendet um die derzeit inaktiven Projektile zu zählen. Der count wird in der `spawn_projectiles()` Funktion genutzt um nur dann Projektile zu spawnen, wenn mindestens so viele Projektile inaktiv und somit vorhanden sind wie die Anzahl die verschossen werden soll.

---

    func get_pooled_projectile():
    	for projectile in projectile_pool:
    		if not projectile.is_active:
    			return projectile
    	return null

Die `get_pooled_projectile()` Funktion wird auch in `spawn_projectiles()` verwendet. Sie wird in jedem Schleifendurchgang von `spawn_projectiles()` aufgerufen und gibt, sofern vorhanden, ein inaktives Projektil zurück. Sind alle Projektile derzeit aktiviert gibt die Funktion `null` zurück.

---

    func spawn_projectiles():
    	if count_inactive_projectiles() < projectiles_per_shot:
    		print("Nicht genügend inaktive Projektile! Benötigt:", projectiles_per_shot)
    		return
    
    	var base_direction = -global_transform.basis.z
    	for i in range(projectiles_per_shot):
    		var projectile = get_pooled_projectile()
    		if projectile:
    			var rotation_transform = Transform3D().rotated(Vector3.UP, 0)
    			var spawn_direction = rotation_transform.basis * base_direction
    			projectile.speed = projectile_speed
    			projectile.activate(spawn_point.global_position, spawn_direction)

Die `spawn_projectiles()` Funkion überprüft zuerst ob genug inaktive Projektile vorhanden. Wenn ja wird `get_pooled_projectile()` aufgerufen um ein Projektil zu erhalten das aktiviert werden soll. 
Wenn das Projektil valide ist werden anschließend `transform` und `rotation` sowie die Geschwindigkeit gesetzt. Anschließend wird die `activate()` Funktion der Projektile aufgerufen welche die zuvor gesetzten Parameter erhält.


</details>

---

<br>

## Beispielprojekt
<br>

|[Hier klicken und ausprobieren](https://kevinryborz.github.io/GodotObjectPooling/)|
|---|
> [!NOTE]
> Mit der **Linken Maustaste** können die Projektile abgefeuert werden. Sind keine im Pool vorhanden werden keine mehr instanziert.
> Alle zerstörten Projektile (aufgrund von Kollision oder Zeit) werden in den Pool zurückgeführt.

|[Project Files](https://github.com/KevinRyborz/GodotObjectPooling/releases/tag/v.0.0.1)|
|---|
