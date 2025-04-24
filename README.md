# Object Pooling in Godot
> [!NOTE]
> [<kbd><br>Web Build</br><br></kbd>](https://kevinryborz.github.io/GodotObjectPooling/)


- [x] #739
- [ ] https://github.com/octo-org/octo-repo/issues/740
- [ ] Add delight to the experience when all tasks are complete :tada:

> [!NOTE]
> Useful information that users should know, even when skimming content.
> ### Erklärung zu `turret.gd`


> [!TIP]
> Helpful advice for doing things better or more easily.

> [!wichtig]
> Key information users need to know to achieve their goal.

> [!WARNING]
> Urgent info that needs immediate user attention to avoid problems.

> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.



## Motivation
In der Spieleentwicklung, insbesondere bei Spielen mit vielen kurzlebigen Objekten (z. B. Projektile, Partikeleffekte, Gegner), kann das ständige Erzeugen (Instanziieren) und Zerstören dieser Objekte zu Leistungseinbußen führen. Jedes instanzierte Objekt wird im Speicher allokiert, initialisiert, zur Szene hinzugefügt und später aufgrund von Ereignissen wie Kollisionen aus dem Speicher entfernt. Wenn dies häufig in kurzer Zeit geschieht, kann es zu spürbaren Rucklern kommen.
Das Object Pooling Pattern ist ein Entwurfsmuster, das dieses Problem minimiert. Anstatt Objekte bei Bedarf neu zu erstellen und später zu zerstören, wird eine vorgegebene Anzahl von Objekten initial erstellt und in einem "Pool" vorgehalten. Wenn ein Objekt benötigt wird, wird es dem Pool entnommen, aktiviert, verwendet und anschließend nicht zerstört, sondern deaktiviert und wieder dem Pool hinzugefügt. Es kann dann erneut aus dem Pool entnommen und wiederverwendet werden.
Stell dir das wie eine Bibliothek vor: Anstatt jedes Mal ein neues Buch zu kaufen (instanziieren) und es nach dem Lesen wegzuwerfen (freigeben), leihst du dir ein Buch aus der Bibliothek (Pool). Nach dem Lesen gibst du es zurück, sodass es für andere verfügbar ist. Die Bibliothek verwaltet die vorhandenen Bücher und gibt sie bei Bedarf aus.
<br>
<br>
## Das Problem
Ein klassisches Beispiel sind Projektile in Bullet Hell Shootern. In jedem Frame oder in kurzen Abständen werden neue Projektile instanziiert. Wenn sie den Bildschirm verlassen oder ein Ziel treffen, werden sie freigegeben. Bei einer großen Anzahl von Projektilen kann dieser Prozess die CPU belasten und zu Leistungsproblemen führen.
Ohne Object Pooling könnte ein typischer Anwendungsfall in Godot so aussehen:

---
1. Ein Bedarf entsteht (z. B. der Spieler schießt).
2. Ein neues Objekt (z. B. ein Projektil) wird instanziiert `(PackedScene.instance())`.
3. Das Objekt wird initialisiert und zum Szenenbaum hinzugefügt `(add_child())`.
4. Das Objekt führt seine Logik aus (bewegt sich, prüft Kollisionen).
5. Das Objekt hat seinen Zweck erfüllt oder ist nicht mehr relevant (trifft etwas, verlässt den Bildschirm).
6. Das Objekt wird zur Freigabe vorgemerkt `(queue_free())`.
7. Später im Frame (oder danach) wird der Speicher des Objekts freigegeben.
---

Dieser ständige Zyklus des Erzeugens und Zerstörens kann erhebliche Verarbeitungszeit beanspruchen und zu Spitzen in der Frametime führen, wenn viele Objekte gleichzeitig erstellt oder freigegeben werden.

> [!IMPORTANT]
> Obwohl Godot's Speicherverwaltung, insbesondere in GDScript, oft effizienter ist als in Umgebungen mit traditioneller Garbage Collection, und `queue_free()` asynchron arbeitet, kann der Overhead des Instanziierens von Szenen immer noch relevant sein, wenn es exzessiv betrieben wird.
<br>


## Wann sollte Object Pooling verwendet werden?
Object Pooling ist sinnvoll, wenn:

- Häufiges Erzeugen und Zerstören von Objekten stattfindet.
- Hohe Kosten für die Objekterzeugung durch Ressourcenallokierung, Verbindungsaufbau oder komplexe Initialisierung entstehen.
- Begrenzte Anzahl gleichzeitig benötigter Objekte existiert, und die maximale Anzahl eingeschränkt werden kann.
- Vorhersehbare Objektlast vorliegt, sodass die ungefähre Anzahl der benötigten Objekte im Voraus abgeschätzt werden kann.

<br>
<br>

## Das Pattern
Das Object Pooling Pattern löst dieses Problem, indem es den Lebenszyklus der Objekte verändert:

---
1. **Initialisierung**: Zu Beginn eines Levels oder Spielabschnitts wird eine bestimmte Anzahl von Objekten des gewünschten Typs erstellt und in einem Pool gespeichert. Diese Objekte sind zunächst inaktiv, unsichtbar und befinden sich in einem deaktivierten Zustand.
2. **Anforderung**: Wenn ein Objekt benötigt wird, fragt der anfordernde Code (der "Client", z. B. der Spieler-Charakter, der ein Projektil abfeuern möchte) den Pool nach einem verfügbaren Objekt.
3. **Entnahme**: Der Pool liefert ein ungenutztes Objekt aus seiner Sammlung.
4. **Nutzung**: Der Client initialisiert das erhaltene Objekt für seine spezifische Verwendung (z. B. setzt Position, Richtung, Geschwindigkeit) und aktiviert es (macht es sichtbar).
5. **Rückgabe**: Wenn das Objekt nicht mehr benötigt wird (z. B. das Projektil trifft ein Ziel oder verlässt den Bildschirm), ruft der Client eine Methode am Pool auf, um das Objekt zurückzugeben.
6. **Wiederverwertung**: Der Pool setzt das zurückgegebene Objekt in seinen ungenutzten Zustand zurück (macht es unsichtbar, deaktiviert es) und fügt es wieder seiner Sammlung hinzu, bereit für die nächste Anforderung.
---

Der Kern des Patterns ist die Vermeidung des wiederholten Instanziierens und Freigebens zur Laufzeit. Stattdessen wird der Hauptaufwand auf die Initialisierung des Pools verlagert. Während des Spiels werden Objekte lediglich "ausgeliehen" und "zurückgegeben", was in der Regel deutlich performanter ist als das vollständige Erstellen und Zerstören von Knoten im Szenenbaum.
<br>
<br>
## Implementierung in Godot
Die Implementierung eines Object Pools in Godot kann auf verschiedene Arten erfolgen, oft unter Verwendung von Arrays oder Dictionaries, um die Objekte zu speichern. Für dieses Beispiel konzentrieren wir uns auf zwei Skripte: projectile.gd für die Funktionalität des Projektils und turret.gd, der den Pool verwaltet und Projektile abfeuert.
Object Pool Skript
Nachfolgend ein Beispiel für ein GDScript zur Verwaltung eines Object Pools:
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
    			var angle_step = 0.0
    			if projectiles_per_shot > 1:
    				angle_step = spread_angle_degrees / (projectiles_per_shot - 1)
    			var current_angle = -spread_angle_degrees / 2.0 + i * angle_step
    			var angle_rad = deg_to_rad(current_angle)
    			var rotation_transform = Transform3D().rotated(Vector3.UP, angle_rad)
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
In der `ready` Funktion werden anschließend mittels eines `for` Loops die Projektile instanziert, als children zum Turret hinzugefügt und deaktiviert um Kollisionen zu verhindern und sie unsichtbar zu machen. In jedem Loop durchlauf werden außerdem mit `projectile_pool.append(projectile)` die
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

<br>

## Beispielprojekt
<br>

|[Web Build](https://kevinryborz.github.io/GodotObjectPooling/)|
|---|
> [!NOTE]
> Mit der **Linken Maustaste** können die Projektile abgefeuert werden. Sind keine im Pool vorhanden werden keine mehr instanziert.
> Alle zerstörten Projektile (aufgrund von Kollision oder Zeit) werden in den Pool zurückgeführt.

|[Project Files](https://github.com/KevinRyborz/GodotObjectPooling/releases/tag/v.0.0.1)|
|---|




## Erklärung

Initialisierung: Der Pool wird in _ready() mit pool_size Instanzen der Projektil-Szene gefüllt. Objekte werden unsichtbar gemacht und deaktiviert, um die Ressourcennutzung zu minimieren.
Objekte anfordern: Die Methode get_object() prüft, ob ein ungenutztes Objekt (basierend auf Sichtbarkeit) verfügbar ist. Falls nicht, wird dynamisch ein neues erstellt.
Objekte zurückgeben: Die Methode return_object() deaktiviert das Objekt und setzt es in einen ungenutzten Zustand zurück, sodass es für zukünftige Anforderungen verfügbar ist.


