<!-- ゲーム画面表示、ゲーム内データストリーム管理 -->
<script setup lang="ts">
import { onMounted, defineProps, ref } from "vue";
import hud_view from "./hud_view.vue";
import type { RT, PT } from "@/@types/types";
import * as THREE from "three";
import { GLTFLoader, GLTF } from "three/examples/jsm/loaders/GLTFLoader.js";
import { Socket } from "socket.io-client";
let width = window.innerWidth;
let height = window.innerHeight;
let mouse_x = 0;
let mouse_y = 0;
const models_dic = {
  desert_eagle: "assets/desert_eagle_reload_animation.glb",
  fn_fal: "assets/fn_fal_reload_animation.glb",
  g3: "assets/g3_reload_animation.glb",
};
const loaded = ref(false);
const props = defineProps<{
  socket: Socket;
  in_nowRT: RT;
  id: string;
}>();
const in_nowRT = ref(props.in_nowRT);
const hud_rendingPT = ref<PT>();
const hud_rending_ammo = ref(0);
const hud_reloading_progress = ref(0);
const debug_data = ref<(string | number)[][]>([
  ["time_delta", 1],
  ["syncT time delta", 1],
  ["npc 9 HP", 1],
]);

onMounted(async () => {
  console.log(in_nowRT);
  let wasd_down = 0;
  let t_down = false;
  let mouse_down = false;
  const clock = new THREE.Clock();
  const move_scale = 850;
  let PCs: PCins[] = [];
  let nPCs: PCins[] = [];
  let bullets: bullet_ins[] = [];

  window.addEventListener("resize", () => {
    width = window.innerWidth;
    height = window.innerHeight;
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(width, height);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
  });
  document.body.addEventListener("keydown", (event) => {
    if (
      PCs &&
      PCs.every((PC) => {
        PC.model;
      })
    )
      return -1;
    if (event.key === "w") {
      wasd_down |= 0b1000;
    }
    if (event.key === "a") {
      wasd_down |= 0b0100;
    }
    if (event.key === "s") {
      wasd_down |= 0b0010;
    }
    if (event.key === "d") {
      wasd_down |= 0b0001;
    }
    if (event.key === "t") {
      t_down = true;
    }
    if (event.key === "r") {
      if (PCs[myPCind]) PCs[myPCind].reload();
    }
    if (event.key === "z") {
      if (PCs[myPCind]) PCs[myPCind].PT.sitting = !PCs[myPCind].PT.sitting;
    }
    if (event.shiftKey) {
      if (PCs[myPCind]) PCs[myPCind].PT.running = !PCs[myPCind].PT.running;
    }
  });
  document.body.addEventListener("keyup", (event) => {
    if (event.key === "w") {
      wasd_down &= 0b0111;
    }
    if (event.key === "a") {
      wasd_down &= 0b1011;
    }
    if (event.key === "s") {
      wasd_down &= 0b1101;
    }
    if (event.key === "d") {
      wasd_down &= 0b1110;
    }
    if (event.key === "t") {
      t_down = false;
    }
  });
  document.body.addEventListener("mousedown", () => {
    mouse_down = true;
  });
  document.body.addEventListener("mouseup", () => {
    mouse_down = false;
    if (PCs[myPCind]) PCs[myPCind].rensha_cool_time = 0;
  });

  document.addEventListener("mousemove", (event) => {
    mouse_x = event.pageX / width - 0.5;
    mouse_y = event.pageY / height - 0.5;
  });
  document.addEventListener("mouseleave", () => {
    mouse_x = 0;
    mouse_y = 0;
  });

  props.socket.on("syncT", (arg: { PTs: PT[]; nPTs: PT[]; time: number }) => {
    debug_data.value[1][1] = (Date.now() - arg.time) / 1000;
    if (PCs) {
      arg.PTs.forEach((newPT, PCind) => {
        if (PCind != myPCind) {
          if (PCind < PCs.length) PCs[PCind].PT = newPT;
          else PCs.push(new PCins(newPT));
        }
      });
      if (arg.nPTs.length < nPCs.length) {
        const removed = nPCs.splice(0, nPCs.length - arg.nPTs.length);
        removed.forEach((nPC) => {
          if (nPC.model) scene.remove(nPC.model);
        });
      }
      arg.nPTs.forEach((new_nPT, nPCind) => {
        if (nPCind % PCs.length !== myPCind) nPCs[nPCind].PT = new_nPT;
      });
    }
  });
  props.socket.on(
    "fire",
    (data: { T: PT; weapon_id: string; ammo_is: boolean }) => {
      const PCind = PCs.findIndex((PC) => PC.PT.id === data.T.id);
      if (PCind !== -1) PCs[PCind].PT = data.T;
      PCs[PCind].shot(data.weapon_id, data.ammo_is);
    }
  );
  props.socket.on("spawn", (PT: PT) => {
    const PCind = PCs.findIndex((PC) => PC.PT.id === PT.id);
    if (PCind !== -1 && PCind !== myPCind) PCs[PCind].PT = PT;
    PCs[PCind].spawn();
  });

  // プレイヤーキャラクターインスタンス
  class PCins {
    PT: PT;
    semi_auto: boolean;
    reload_time: number;
    reloading_time: number;
    rensha: number;
    rensha_cool_time: number;
    max_ammo: number;
    ammo: number;
    damage: number;
    bul_speed: number;
    repopcount: number;
    shot_sound: HTMLAudioElement;
    reload_sound: HTMLAudioElement;
    noshot_sound: HTMLAudioElement;
    model_path: string;
    gltf: GLTF | undefined;
    model: THREE.Object3D | undefined;
    mixer: THREE.AnimationMixer | undefined;
    animations: Map<string, THREE.AnimationClip>;
    now_animation: THREE.AnimationClip | undefined;
    action: THREE.AnimationAction | undefined;
    constructor(PT: PT) {
      this.PT = PT;
      switch (this.PT.weapon_ids.main) {
        case "desert_eagle": // デザートイーグル
          this.rensha = 1;
          this.max_ammo = 7;
          this.damage = 3;
          this.bul_speed = 460;
          this.semi_auto = true;
          this.reload_time = 2.5;
          break;
        case "fn_fal": // fal
          this.rensha = 0.091;
          this.max_ammo = 20;
          this.damage = 0.6;
          this.bul_speed = 823;
          this.semi_auto = false;
          this.reload_time = 2.0;
          break;
        case "g3": // G3
          this.rensha = 0.1;
          this.max_ammo = 30;
          this.damage = 0.5;
          this.bul_speed = 790;
          this.semi_auto = false;
          this.reload_time = 5.5;
          break;
      }
      this.model = undefined;
      this.shot_sound = new Audio("assets/submachinegun.mp3");
      this.reload_sound = new Audio("assets/pompaction.mp3");
      this.noshot_sound = new Audio("assets/pompaction_none.mp3");
      this.model_path = models_dic[this.PT.weapon_ids.main];
      this.animations = new Map([]);
      this.reloading_time = 0;
      this.rensha_cool_time = 0;
      this.repopcount = 2;
      this.ammo = 0;
      this.mixer = undefined;
      this.init();
    }

    async init() {
      this.gltf = await loadergltf.loadAsync(this.model_path);
      this.model = this.gltf.scene;
      if (!this.model) throw Error(this.model_path + "ロード失敗");
      if (this.gltf.animations.length > 0) {
        this.gltf.animations.forEach((clip: THREE.AnimationClip) => {
          // アニメーションをnameインデックスに
          this.animations.set(clip.name, clip);
        });
        this.mixer = new THREE.AnimationMixer(this.model);
      }
      scene.add(this.model);
      if (
        this.PT.weapon_ids.main === "desert_eagle" ||
        this.PT.weapon_ids.main === "g3"
      ) {
        this.model.scale.set(1.08, 1.08, 1.08);
      } else if (this.PT.weapon_ids.main === "fn_fal") {
        this.model.scale.set(91.8, 91.8, 91.8);
      }
      console.log(this.PT.id, this.animations.keys(), this.model);
      if (props.id === this.PT.id) this.spawn();
      else this.update(0.01);
    }

    reload() {
      if (this.reloading_time <= 0) {
        this.anim_change("reload");
        this.reloading_time = this.reload_time;
        if (this.shot_sound.readyState === 4) {
          this.reload_sound.pause();
          this.reload_sound.currentTime = 0;
          this.reload_sound.play();
        }
        this.ammo = this.max_ammo;
      }
    }

    update(time_delta: number) {
      if (!this.model) return -1;
      if (this.repopcount >= 0) this.repopcount -= time_delta;
      if (this.rensha_cool_time >= 0) this.rensha_cool_time -= time_delta;
      if (this.reloading_time >= 0) this.reloading_time -= time_delta;
      let poseflag = true;
      if (this.reloading_time >= 0) poseflag = false;

      if (this.PT.sitting && poseflag) {
        this.anim_change("sit");
        poseflag = false;
        // sitとrunは未実装。以下のoriginコードを参照に。
        //   if (this.statuses & 0b1000) this.nowspeed += (2 - this.nowspeed) * 0.1;
        //   else this.nowspeed += (0.8 - this.nowspeed) * 0.2;
        // } else {
        //   this.nowspeed += (0.3 - this.nowspeed) * 0.03;
      }

      if (
        (this.repopcount >= 0 &&
          Math.ceil(this.repopcount ** 2 / 5 + this.repopcount * 5) % 2) ||
        !this.PT.alive
      )
        this.model.visible = false;
      else this.model.visible = true;

      this.PT.position.x += this.PT.velocity.x * time_delta;
      this.PT.position.y += this.PT.velocity.y * time_delta;
      this.PT.position.z += this.PT.velocity.z * time_delta;
      this.model.position.set(
        this.PT.position.x,
        this.PT.position.y,
        this.PT.position.z
      );
      PCs[myPCind].PT.velocity.x *= 0.8;
      PCs[myPCind].PT.velocity.z *= 0.8;
      if (
        this.PT.velocity.x ** 2 +
          this.PT.velocity.y ** 2 +
          this.PT.velocity.z ** 2 >
          0.1 &&
        poseflag
      ) {
        this.anim_change("walk");
        poseflag = false;
      }
      this.PT.position.y_rotation += this.PT.velocity.y_rotation * time_delta;
      this.PT.position.elevation_angle +=
        this.PT.velocity.elevation_angle * time_delta;
      this.model.rotation.set(0, this.PT.position.y_rotation, 0);
      this.PT.velocity.y_rotation /= 0.1;
      this.PT.velocity.elevation_angle /= 0.1;

      if (poseflag) this.anim_change("pose");
      if (this.mixer) this.mixer.update(time_delta);
    }

    shot(weapon_id: string, ammo_is: boolean) {
      if (!loaded.value) return -1;
      if (ammo_is) {
        // this.ammo--;
        bullets.push(new bullet_ins(this.PT));
        if (this.shot_sound.readyState === 4) {
          this.shot_sound.pause();
          this.shot_sound.currentTime = 0;
          this.shot_sound.play();
        }
      } else {
        // 空打ち
        if (this.noshot_sound.readyState === 4) {
          this.noshot_sound.pause();
          this.noshot_sound.currentTime = 0;
          this.noshot_sound.play();
        }
      }
    }

    spawn() {
      this.repopcount = 3;
      this.rensha_cool_time = 2;
      this.PT.alive = true;
      this.PT.health = 10;
      this.PT.death++;
      this.PT.position = this.PT.spawn_point;
      this.PT.sitting = false;
      this.PT.running = false;
    }
    anim_change(mode: string) {
      if (
        (!this.now_animation || this.now_animation.name != mode) &&
        this.mixer
      ) {
        console.log(mode);
        if (mode === "reload" && this.animations.has("KeptYouWaitingHuh?")) {
          // アニメーションを作ってないfal用
          this.mixer.stopAllAction();
          if (this.action) this.action.reset();
          this.now_animation = this.animations.get("KeptYouWaitingHuh?");
          if (!this.now_animation) return -1;
          this.action = this.mixer.clipAction(this.now_animation);
          this.action.setLoop(THREE.LoopOnce, 1);
          this.action.clampWhenFinished = true;
          this.action.play();
        } else {
          this.mixer.stopAllAction();
          if (this.action) this.action.reset();
          this.now_animation = this.animations.get(mode);
          if (!this.now_animation) return -1;
          this.action = this.mixer.clipAction(this.now_animation);
          this.action.setLoop(THREE.LoopRepeat, 999999);
          this.action.clampWhenFinished = true;
          this.action.play();
        }
      }
    }
  }
  function kia_bullet(bullet_id: number) {
    let target_bullet_ind = bullets.findIndex((b) => b.bullet_id === bullet_id);
    if (target_bullet_ind !== -1) {
      scene.remove(bullets[target_bullet_ind].bullet);
      bullets.splice(target_bullet_ind, 1);
    } else {
      console.error("error kia bullet", bullets.length);
    }
  }
  class bullet_ins {
    bullet: THREE.Mesh<
      THREE.BoxGeometry,
      THREE.MeshBasicMaterial,
      THREE.Object3DEventMap
    >;
    bullet_id: number;
    Pside: number;
    vec: { y: number; e: number } | undefined;
    speed: number | undefined;
    counter: number | undefined;
    damage: number | undefined;
    ray: THREE.Raycaster | undefined;
    // material.DepthがTSで表現できないため。
    intersects: any[] | undefined;
    exist: boolean;
    constructor(T: PT) {
      this.bullet = new THREE.Mesh(
        new THREE.BoxGeometry(3, 3, 6),
        new THREE.MeshBasicMaterial({ color: 0xff0000 })
      );
      this.Pside = T.side;
      this.bullet_id = Math.random();
      this.exist = true;
      const PT = T;
      const PC = PCs.find((PC) => PC.PT.id === T.id);
      if (!PC) {
        kia_bullet(this.bullet_id);
        return;
      }
      this.vec = { y: PT.position.y_rotation, e: PT.position.elevation_angle };
      this.bullet.position.set(
        PT.position.x + Math.sin(this.vec.y) * Math.cos(this.vec.e) * 20,
        PT.position.y + 170 + Math.sin(this.vec.e) * 20,
        PT.position.z + Math.cos(this.vec.y) * Math.cos(this.vec.e) * 20
      );
      this.bullet.rotation.set(-this.vec.e, this.vec.y, 0);
      scene.add(this.bullet);

      this.speed = PC.bul_speed * 50;
      // this.speed = PC.bul_speed * 0.5;
      this.counter = 3000;
      this.damage = PC.damage;
    }
    move(time_delta: number) {
      if (!this.counter || !this.speed || !this.vec || !this.damage) return;
      this.counter -= this.speed * time_delta;
      if (this.counter < 0) {
        kia_bullet(this.bullet_id);
        return;
      }

      this.ray = new THREE.Raycaster(
        this.bullet.position,
        new THREE.Vector3(
          Math.sin(this.vec.y) * Math.cos(this.vec.e),
          Math.sin(this.vec.e),
          Math.cos(this.vec.y) * Math.cos(this.vec.e)
        ),
        0,
        this.speed * time_delta
      );
      if (PCs[myPCind].model && this.ray) {
        this.intersects = this.ray.intersectObjects([PCs[myPCind].model]);
        if (this.intersects.length > 0) {
          if (PCs[myPCind].PT.side != this.Pside) {
            // const head_shot = [263, 279, 285, 313];
            const hit_object = this.intersects[0].object;
            // if (head_shot.find((e) => e === hit_object.id)) {
            //   PCs[myPCind].PT.health -= this.damage * 2;
            //   console.log("got head shot", PCs[myPCind].PT.id, this.Pside);
            // } else {
            PCs[myPCind].PT.health -= this.damage;
            if (PCs[myPCind].PT.health <= 0) {
              PCs[myPCind].PT.alive = false;
            }
            console.log(
              "got body shot",
              PCs[myPCind].PT.id,
              hit_object.id,
              this.Pside
            );
            this.exist = false;
          }
        }
      }
      PCs.forEach((PC, i) => {
        if (PC.model && this.ray && i % PCs.length == myPCind) {
          this.intersects = this.ray.intersectObjects([PC.model]);
          if (this.intersects.length > 0) {
            // ダメージ計算は自身のみ
            if (PC.PT.id == props.id && PC.PT.side != this.Pside) {
              // ヘッドショット後回し
              // const head_shot = [263, 279, 285, 313];
              const hit_object = this.intersects[0].object;
              // if (head_shot.find((e) => e === hit_object.id)) {
              //   nPCs[i].PT.health -= this.damage * 2;
              //   console.log("got head shot", nPC.PT.id);
              // } else {
              nPCs[i].PT.health -= this.damage as number;
              console.log("got body shot", PC.PT.id, hit_object.id);
              hit_object.material.color.r = 1;
              this.exist = false;
            }
          }
        }
      });
      nPCs.forEach((nPC, i) => {
        // nPCの処理担当はインしているPCに均等に。
        // あと処理が深すぎるせいかeslintがdamage疑ってるのでas。
        if (nPC.model && this.ray && i % PCs.length == myPCind) {
          this.intersects = this.ray.intersectObjects([nPC.model]);
          if (this.intersects.length > 0) {
            if (nPC.PT.side != this.Pside) {
              // ヘッドショット後回し
              // const head_shot = [263, 279, 285, 313];
              const hit_object = this.intersects[0].object;
              // if (head_shot.find((e) => e === hit_object.id)) {
              //   nPCs[i].PT.health -= this.damage * 2;
              //   console.log("got head shot", nPC.PT.id);
              // } else {
              nPCs[i].PT.health -= this.damage as number;
              console.log("got body shot", nPC.PT.id, hit_object.id);
              hit_object.material.color.r = 1;
              this.exist = false;
            }
          }
        }
      });
      if (!this.exist) {
        kia_bullet(this.bullet_id);
      }

      this.bullet.position.x +=
        Math.sin(this.vec.y) * Math.cos(this.vec.e) * this.speed * time_delta;
      this.bullet.position.y += Math.sin(this.vec.e) * this.speed * time_delta;
      this.bullet.position.z +=
        Math.cos(this.vec.y) * Math.cos(this.vec.e) * this.speed * time_delta;
    }
  }
  function camset_fps(target_id: number) {
    const vec = {
      y: PCs[target_id].PT.position.y_rotation,
      e: PCs[target_id].PT.position.elevation_angle,
    };
    camera.position.set(
      PCs[target_id].PT.position.x + Math.sin(vec.y) * Math.cos(vec.e) * 37,
      PCs[target_id].PT.position.y + 175 + Math.sin(vec.e) * 37,
      PCs[target_id].PT.position.z + Math.cos(vec.y) * Math.cos(vec.e) * 37
    );
    camera.lookAt(
      PCs[target_id].PT.position.x + Math.sin(vec.y) * Math.cos(vec.e) * 40,
      PCs[target_id].PT.position.y + 175 + Math.sin(vec.e) * 40,
      PCs[target_id].PT.position.z + Math.cos(vec.y) * Math.cos(vec.e) * 40
    );
  }
  // function camset_test(targetPCind: number) {
  //   camera.position.set(
  //     PCs[targetPCind].PT.position.x,
  //     PCs[targetPCind].PT.position.y + 100,
  //     PCs[targetPCind].PT.position.z + 500
  //   );
  //   camera.lookAt(
  //     PCs[targetPCind].PT.position.x,
  //     PCs[targetPCind].PT.position.y + 100,
  //     PCs[targetPCind].PT.position.z
  //   );
  // }

  const canvas = document.getElementById("main_canvas");
  if (canvas === null) throw Error("main_canvas検知不能");
  const renderer = new THREE.WebGLRenderer({
    canvas,
    logarithmicDepthBuffer: true,
  });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(width, height);
  const scene = new THREE.Scene();
  const loadergltf = new GLTFLoader();

  const camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 200000);

  const light = new THREE.HemisphereLight(0x888888, 0x505000, 1.0);
  scene.add(light);

  let model_r: THREE.Group<THREE.Object3DEventMap>;
  loadergltf.load("assets/de_dust2_-_cs_map-rep.glb", function (gltf) {
    model_r = gltf.scene;
    scene.add(model_r);
    model_r.scale.set(250, 250, 250);
    model_r.position.set(0, 0, 0);
  });

  in_nowRT.value.PTs.forEach((PT) => {
    PCs.push(new PCins(PT));
  });
  in_nowRT.value.nPTs.forEach((nPT) => {
    nPCs.push(new PCins(nPT));
  });
  const myPCind = PCs.findIndex((PC) => PC.PT.id === props.id);

  tick();
  function tick() {
    if (
      PCs.findIndex((PC) => {
        PC.model;
      }) === -1 &&
      nPCs.findIndex((PC) => {
        PC.model;
      }) === -1 &&
      model_r
    ) {
      loaded.value = true;
    }
    let time_delta = clock.getDelta();
    debug_data.value[0][1] = Math.floor(time_delta * 1000) / 1000;
    const nPC9 = nPCs.find((nPC) => nPC.PT.id === "npc_9");
    if (nPC9) {
      debug_data.value[2][1] = nPC9.PT.health;
    }

    PCs.forEach((PC) => {
      PC.update(time_delta);
    });
    nPCs.forEach((nPC) => {
      nPC.update(time_delta);
    });
    bullets.forEach((bullet) => {
      bullet.move(time_delta);
    });
    if (wasd_down > 0) {
      let move_rad: number;
      if (wasd_down === 8) move_rad = 0;
      else if (wasd_down === 12) move_rad = 0.25;
      else if (wasd_down === 4) move_rad = 0.5;
      else if (wasd_down === 6) move_rad = 0.75;
      else if (wasd_down === 2) move_rad = 1;
      else if (wasd_down === 3) move_rad = 1.25;
      else if (wasd_down === 1) move_rad = 1.5;
      else move_rad = 1.75; // 9
      if (PCs[myPCind].model) {
        const myPCmoving_ray = new THREE.Raycaster(
          new THREE.Vector3(
            PCs[myPCind].PT.position.x,
            PCs[myPCind].PT.position.y + 20,
            PCs[myPCind].PT.position.z
          ),
          new THREE.Vector3(
            Math.sin(Math.PI * move_rad + PCs[myPCind].PT.position.y_rotation),
            0,
            Math.cos(Math.PI * move_rad + PCs[myPCind].PT.position.y_rotation)
          ),
          0,
          move_scale * time_delta + 50
        );
        const intersects = myPCmoving_ray.intersectObjects([model_r]);
        if (intersects.length > 0) {
          PCs[myPCind].PT.velocity.x /= 2;
          PCs[myPCind].PT.velocity.z /= 2;
        } else {
          PCs[myPCind].PT.velocity.x =
            Math.sin(Math.PI * move_rad + PCs[myPCind].PT.position.y_rotation) *
            move_scale;
          PCs[myPCind].PT.velocity.z =
            Math.cos(Math.PI * move_rad + PCs[myPCind].PT.position.y_rotation) *
            move_scale;
        }
      }
    } else {
      PCs[myPCind].PT.velocity.x /= 2;
      PCs[myPCind].PT.velocity.z /= 2;
    }
    PCs[myPCind].PT.position.y_rotation -= mouse_x * 0.06;
    PCs[myPCind].PT.position.elevation_angle -= mouse_y * 0.06;
    if (PCs[myPCind].PT.position.elevation_angle > 0.125 * Math.PI)
      PCs[myPCind].PT.position.elevation_angle = 0.125 * Math.PI;
    if (PCs[myPCind].PT.position.elevation_angle < -0.03125 * Math.PI)
      PCs[myPCind].PT.position.elevation_angle = -0.03125 * Math.PI;
    props.socket.emit("syncT", { time: Date.now(), PT: PCs[myPCind].PT });
    nPCs.forEach((nPC, i) => {
      if (i % PCs.length === myPCind) {
        props.socket.emit("npc_syncT", nPC.PT);
      }
    });
    if (
      mouse_down &&
      PCs[myPCind].reloading_time < 0 &&
      PCs[myPCind].rensha_cool_time < 0
    ) {
      props.socket.emit("fire", {
        T: PCs[myPCind].PT,
        weapon_id: PCs[myPCind].PT.weapon_ids.main,
        ammo_is: PCs[myPCind].ammo > 0,
      });
      PCs[myPCind].rensha_cool_time = PCs[myPCind].rensha;
      PCs[myPCind].ammo--;
    }
    if (t_down && !PCs[myPCind].PT.alive) {
      PCs[myPCind].PT.position = PCs[myPCind].PT.spawn_point;
      props.socket.emit("spawn", PCs[myPCind].PT);
    }
    hud_rendingPT.value = PCs[myPCind].PT;
    hud_rending_ammo.value = PCs[myPCind].ammo;
    hud_reloading_progress.value =
      PCs[myPCind].reloading_time / PCs[myPCind].reload_time;
    // fovcos += 0.01;
    // camera.fov = (Math.cos(fovcos) + 1) * 5 + 60;
    // camera.updateProjectionMatrix();
    // camset_test(myPCind);
    camset_fps(myPCind);
    renderer.render(scene, camera); // レンダリング
    requestAnimationFrame(tick);
  }
});
</script>

<style scoped>
#yet_load {
  position: absolute;
  z-index: 2;
  left: 20%;
  top: 20%;
}
#yet_load img {
  width: 60%;
}
canvas {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 0;
  overflow: hidden;
  user-select: none;
}
</style>

<template>
  <div v-if="!loaded" id="yet_load">
    <img src="assets/ozisanoffense_logo.png" alt="title_wall" />
    <p>ロード中</p>
  </div>
  <hud_view
    v-if="hud_rendingPT"
    :hud_data="{
      PT: hud_rendingPT,
      ammo: hud_rending_ammo,
      reloading_progress: hud_reloading_progress,
    }"
    :debug_data="debug_data"
  />
  <canvas id="main_canvas" />
</template>
