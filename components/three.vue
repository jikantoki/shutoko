<template lang="pug">
  div(
    ref="container"
  )
</template>

<script>
import * as THREE from 'three'
import * as CANNON from 'cannon-es'
import { damp } from 'three/src/math/MathUtils'

export default {
  data() {
    return {
      container: null,
    }
  },
  mounted() {
    const scene = new THREE.Scene()
    const camera = new THREE.PerspectiveCamera()
    const renderer = new THREE.WebGLRenderer({
      antialias: true,
    })

    // CANNON.jsの初期化
    const world = new CANNON.World()
    world.gravity.set(0, -9.82, 0) // 重力を設定

    /** 物体のリスト */
    const objectList = []

    /** 地面を作成 */
    const ground = {
      name: 'ground',
      mass: 0, // 地面は動かないので質量は0
      position: new CANNON.Vec3(0, 0, 0),
      shape: new CANNON.Box(new CANNON.Vec3(10, 0.01, 10)), // 地面の形状
      material: new CANNON.Material({
        restitution: 0.5, // 反発係数
      }),
      mesh: new THREE.Mesh(
        new THREE.BoxGeometry(20, 0.02, 20),
        new THREE.MeshStandardMaterial({
          color: 0x00ff00,
          transparent: true,
          opacity: 0.5,
        }),
      ),
    }
    objectList.push(ground)

    /** 球体を作成 */
    const Sphere = {
      name: 'sphere',
      mass: 10, // 質量を設定
      position: new CANNON.Vec3(0, 10, 0), // 初期位置を設定
      shape: new CANNON.Sphere(0.5), // 球体の半径を0.5に設定
      material: new CANNON.Material({
        restitution: 1, // 反発係数
      }),
      mesh: new THREE.Mesh(
        new THREE.SphereGeometry(0.5, 32, 32),
        new THREE.MeshStandardMaterial({
          color: 0xff9999,
          transparent: true,
          opacity: 0.75,
        }),
      ),
    }
    objectList.push(Sphere)

    /** 車体のボディー */
    const car = {
      name: 'car',
      vehicle: true,
      mass: 1000, // 車の質量を設定
      position: new CANNON.Vec3(0, 4.5, 5),
      shape: new CANNON.Box(new CANNON.Vec3(3.8, 2.5, 1.4)), // 車の形状を箱型に設定
      mesh: new THREE.Mesh(
        new THREE.BoxGeometry(3.8, 2.5, 1.4),
        new THREE.MeshStandardMaterial({
          color: 0x0000ff,
          transparent: true,
          opacity: 0.75,
        }),
      ),
      wheelOptions: {
        radius: 0.5, // 車輪の半径を0.5に設定
        directionLocal: new CANNON.Vec3(0, 0, -1), // 車輪の方向を設定
        suspensionStiffness: 50, // サスペンションの剛性を設定
        suspensionRestLength: 0.5, // サスペンションの初期長さを設定
        frictionSlip: 5, // 摩擦係数を設定
        dampingRelaxation: 2.3, // ダンピングの緩和を設定
        dampingCompression: 4.4, // 跳ね返りの強さを設定
        maxSuspensionForce: 100000, // 最大サスペンション力を設定
        rollInfluence: 0.01, // 車輪の転がり影響を設定
        axleLocal: new CANNON.Vec3(0, 1, 0), // 車輪の軸を設定
        chassisConnectionPointLocal: new CANNON.Vec3(1, 1, 0), // 車体との接続点を設定
        maxSuspensionTravel: 0.3, // 最大サスペンションの移動距離を設定
        customSlidingRotationalSpeed: -30, // 車輪の回転速度を設定
        useCustomSlidingRotationalSpeed: true, // カスタム回転速度を使用
      },
    }
    objectList.push(car)

    //objectListから生成した、bodyとmeshのリスト
    const bodyMeshList = []

    let i = 0
    for (const object of objectList) {
      /** CANNON.jsの物理ボディを作成 */
      const body = this.createBody(object)

      /** THREE.jsのメッシュを作成 */
      const mesh = object.mesh
      mesh.position.copy(body.position)
      mesh.quaternion.copy(body.quaternion)

      /** ホイール情報リスト */
      const wheelInfoArray = []

      //車両オブジェクトの場合の設定
      if (object.vehicle) {
        const raycastVehicle = new CANNON.RaycastVehicle({
          chassisBody: body, // 車体の物理ボディを設定
        })
        object.wheelOptions.chassisConnectionPointLocal.set(1, 1, 0)
        raycastVehicle.addWheel(object.wheelOptions)
        object.wheelOptions.chassisConnectionPointLocal.set(1, -1, 0)
        raycastVehicle.addWheel(object.wheelOptions)
        object.wheelOptions.chassisConnectionPointLocal.set(-1, 1, 0)
        raycastVehicle.addWheel(object.wheelOptions)
        object.wheelOptions.chassisConnectionPointLocal.set(-1, -1, 0)
        raycastVehicle.addWheel(object.wheelOptions)

        raycastVehicle.addToWorld(world) // RaycastVehicleを物理世界に追加

        //ホイールを1つずつ追加
        for (let j = 0; j < raycastVehicle.wheelInfos.length; j++) {
          const wheel = raycastVehicle.wheelInfos[j]
          const wheelBody = new CANNON.Body({
            mass: 1, // 車輪の質量を設定
          })
          const q = new CANNON.Quaternion() // 車輪の回転を設定するためのクォータニオン
          q.setFromAxisAngle(new CANNON.Vec3(1, 0, 0), Math.PI / 2) // 車輪を横向きにするための回転
          wheelBody.addShape(
            new CANNON.Cylinder(
              wheel.radius,
              wheel.radius,
              wheel.radius, // 車輪の高さを設定
              20, // サイドの数を設定
            ),
            new CANNON.Vec3(), // 車輪の位置を設定
            q, // 車輪の回転を設定
          )
          const wheelMesh = new THREE.Mesh(
            new THREE.CylinderGeometry(
              wheel.radius,
              wheel.radius,
              wheel.radius, // 車輪の高さを設定
              20, // サイドの数を設定
            ),
            new THREE.MeshStandardMaterial({
              color: 0x000000,
              transparent: true,
              opacity: 0.75,
            }),
          )
          wheelInfoArray.push({
            body: wheelBody,
            mesh: wheelMesh,
          })
          scene.add(wheelMesh)
        }

        // 車両のホイールの位置と回転を更新
        world.addEventListener('postStep', () => {
          for (let j = 0; j < wheelInfoArray.length; j++) {
            raycastVehicle.updateWheelTransform(j)
            const t = raycastVehicle.wheelInfos[j].worldTransform
            const info = wheelInfoArray[j]
            info.body.position.copy(t.position)
            info.body.quaternion.copy(t.quaternion)
          }
        })
      }

      bodyMeshList.push({
        body: body,
        mesh: mesh,
        isVehicle: object.vehicle,
        wheelInfoArray: object.vehicle ? wheelInfoArray : null,
      })
      world.addBody(bodyMeshList[i].body)
      scene.add(bodyMeshList[i].mesh)

      i++
    }

    const copy = () => {
      // CANNON.jsの物理ボディの位置と回転をTHREE.jsのメッシュにコピー
      for (const bodyMesh of bodyMeshList) {
        bodyMesh.mesh.position.copy(bodyMesh.body.position)
        bodyMesh.mesh.quaternion.copy(bodyMesh.body.quaternion)
        if (bodyMesh.isVehicle) {
          // 車両のホイールの位置と回転を更新
          for (let j = 0; j < bodyMesh.wheelInfoArray.length; j++) {
            const info = bodyMesh.wheelInfoArray[j]
            info.mesh.position.copy(info.body.position)
            info.mesh.quaternion.copy(info.body.quaternion)
          }
        }
      }
    }

    this.container = this.$refs.container

    /** 環境光源 */
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
    directionalLight.position.set(0, 1, 0)
    scene.add(directionalLight)

    this.initThree(scene, camera, renderer, copy, world)
  },
  methods: {
    /**
     * CANNON.jsの物理ボディを作成
     */
    createBody(object) {
      const body = new CANNON.Body({
        mass: object.mass,
        position: object.position,
        shape: object.shape,
        material: object.material,
      })

      return body
    },
    /**
     * 世界を生成
     */
    initThree(scene, camera, renderer, copy, world) {
      //DOMサイズ取得
      const clientWidth =
        window.innerWidth ||
        document.documentElement.clientWidth ||
        document.body.clientWidth
      const clientHeight =
        window.innerHeight ||
        document.documentElement.clientHeight ||
        document.body.clientHeight

      //グリッド追加
      scene.add(new THREE.GridHelper(20, 20, 0x0000ff, 0x404040))

      //カメラ設定
      camera.aspect = clientWidth / clientHeight
      camera.updateProjectionMatrix()
      camera.position.set(30, 10, 0)
      camera.lookAt(0, 3, 0)

      //レンダラー設定
      renderer.setSize(clientWidth, clientHeight)
      renderer.setPixelRatio(clientWidth / clientHeight)
      this.$refs.container.appendChild(renderer.domElement)

      const animate = () => {
        const frame = () => {
          copy()
          world.fixedStep()
          renderer.render(scene, camera)
          requestAnimationFrame(frame)
        }
        frame()
      }

      animate()
    },
  },
}
</script>
