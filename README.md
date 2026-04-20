# kombat-3d
3D fighting game built with Three.js
extends CharacterBody3D
class_name Fighter3D

@export var speed := 5.0
@export var jump_velocity := 8.0
@export var max_health := 100.0
@export var is_player_1 := true

var health := 100.0
var gravity = ProjectSettings.get_setting("physics/3d/default_gravity")

var facing_right := true
var opponent: Fighter3D

enum State { IDLE, WALK, JUMP, ATTACK, HIT, BLOCK, DEAD }
var current_state = State.IDLE
var state_timer := 0.0

@onready var hitbox: Area3D = $Hitbox
@onready var anim: AnimationPlayer = $AnimationPlayer

func _ready():
    add_to_group("fighters")
    find_opponent()
    hitbox.monitoring = false
    health = max_health

func find_opponent():
    for f in get_tree().get_nodes_in_group("fighters"):
        if f != self:
            opponent = f

func _physics_process(delta):
    if current_state == State.DEAD:
        return

    # Gravity
    if not is_on_floor():
        velocity.y -= gravity * delta

    handle_input()
    move_and_slide()

    face_opponent()

func face_opponent():
    if opponent:
        var dir = opponent.global_position - global_position
        facing_right = dir.x > 0
        rotation.y = 0 if facing_right else PI

func handle_input():
    var dir := 0

    if is_player_1:
        if Input.is_action_pressed("move_left"): dir -= 1
        if Input.is_action_pressed("move_right"): dir += 1
        if Input.is_action_just_pressed("jump") and is_on_floor():
            velocity.y = jump_velocity
            anim.play("jump")

        if Input.is_action_just_pressed("punch"):
            attack()

    else:
        if Input.is_action_pressed("p2_left"): dir -= 1
        if Input.is_action_pressed("p2_right"): dir += 1
        if Input.is_action_just_pressed("p2_jump") and is_on_floor():
            velocity.y = jump_velocity
            anim.play("jump")

        if Input.is_action_just_pressed("p2_punch"):
            attack()

    velocity.x = dir * speed

    if dir != 0 and is_on_floor():
        anim.play("walk")
    elif is_on_floor():
        anim.play("idle")

func attack():
    if current_state == State.ATTACK:
        return

    current_state = State.ATTACK
    hitbox.monitoring = true
    anim.play("punch")

    await get_tree().create_timer(0.3).timeout
    hitbox.monitoring = false
    current_state = State.IDLE

func _on_hitbox_body_entered(body):
    if body is Fighter3D and body != self:
        body.take_damage(12)
        body.velocity.x += 4 if facing_right else -4

func take_damage(dmg):
    if current_state == State.DEAD:
        return

    health -= dmg
    anim.play("hit")

    if health <= 0:
        die()

func die():
    current_state = State.DEAD
    anim.play("death")
