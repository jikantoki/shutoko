<template lang="pug">
  div(
    ref="container"
  )
</template>

<script>
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { ThreeJSOverlayView } from '@googlemaps/three'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import * as CANNON from 'cannon-es'

export default {
  data() {
    return {
      container: null,
      cameraPos: {
        x: 0,
        y: 0,
        z: 0,
      },
      cameraLookAt: {
        x: 0,
        y: 0,
        z: 0,
      },
      cameraForward: {
        x: 1,
        y: 1,
        z: 0,
      },
    }
  },
  async mounted() {
    /* キーコードの定義 */
    const key = {
      up: 38,
      down: 40,
      left: 37,
      right: 39,
      w: 87,
      a: 65,
      s: 83,
      d: 68,
      space: 32,
      b: 66,
      q: 81,
      e: 69,
    }

    const scene = new THREE.Scene()
    //scene.background = new THREE.Color(0xeeeeee) // 背景色を設定
    scene.fog = new THREE.FogExp2(0xeeeeee, 0.01) // フォグを設定
    const camera = new THREE.PerspectiveCamera()
    const renderer = new THREE.WebGLRenderer({
      antialias: true, // アンチエイリアスを有効にする
    })
    renderer.shadowMap.enabled = true // 影を有効にする

    const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256)
    cubeRenderTarget.texture.type = THREE.HalfFloatType // 半精度浮動小数点数を使用
    cubeRenderTarget.texture.mapping = THREE.CubeReflectionMapping // キューブマッピングを使用

    const cubeCamera = new THREE.CubeCamera(1, 1000, cubeRenderTarget)

    // CANNON.jsの初期化
    const world = new CANNON.World()
    world.gravity.set(0, -9.82, 0) // 重力を設定
    world.addContactMaterial(
      new CANNON.ContactMaterial(
        new CANNON.Material('wheelMaterial'),
        new CANNON.Material('groundMaterial'),
        {
          friction: 0.3, // 摩擦係数
          restitution: 0, // 反発係数
          contactEquationStiffness: 1000, // 接触方程式の剛性
        },
      ),
    )

    /** 物体のリスト */
    const objectList = []

    /** raycastvehicle型の自分が操縦する車 */
    let myVehicle = null

    /** 地面を作成 */
    const ground = {
      name: 'ground',
      mass: 0, // 地面は動かないので質量は0
      isStatic: true, // 静的な物体として設定
      position: new CANNON.Vec3(0, 0, 0),
      shape: new CANNON.Box(new CANNON.Vec3(30, 0.01, 30)), // 地面の形状
      material: new CANNON.Material({
        restitution: 0.5, // 反発係数
      }),
      mesh: new THREE.Mesh(
        new THREE.BoxGeometry(60, 0.02, 60),
        new THREE.MeshStandardMaterial({
          color: 0x222222,
        }),
      ),
    }
    objectList.push(ground)

    //gridhelper
    const gridHelper = new THREE.GridHelper(60, 60, 0x0000ff, 0x404040)
    gridHelper.position.set(0, 0.01, 0) // 地面
    scene.add(gridHelper)

    /** 球体を作成 */
    const Sphere = () => {
      return {
        name: 'sphere',
        mass: 10, // 質量を設定
        position: new CANNON.Vec3(5, 10, 0), // 初期位置を設定
        shape: new CANNON.Sphere(0.5), // 球体の半径を0.5に設定
        material: new CANNON.Material({
          restitution: 1, // 反発係数
        }),
        mesh: new THREE.Mesh(
          new THREE.SphereGeometry(0.5, 32, 32),
          new THREE.MeshStandardMaterial({
            color: 0xff9999,
            transparent: true,
            roughness: 0.0, // 非光沢度を設定
            metalness: 1.0, // 金属感を設定
            envMap: cubeRenderTarget.texture, // 環境マッピングを設定
          }),
        ),
      }
    }
    objectList.push(Sphere())

    const sphere2 = Sphere()
    sphere2.position.set(-5, 8, 0) // 球体の位置を変更
    sphere2.mass = 0.1 // 球体の質量を変更
    objectList.push(sphere2)

    const carMeshes = new THREE.Object3D() // 車のメッシュを格納するオブジェクト
    const carBody = new THREE.Mesh(
      new THREE.BoxGeometry(3.8, 1.2, 1.6),
      new THREE.MeshStandardMaterial({
        color: 0xffffff,
        roughness: 0.0, // 非光沢度を設定
        metalness: 1, // 金属感を設定
        reflectivity: 1, // 反射率を設定
        transparent: true, // 透明度を有効にする
        envMap: cubeRenderTarget.texture, // 環境マッピングを設定
      }),
    )
    carBody.castShadow = true // 車体が影を落とすように設定

    /** 車体のメッシュを作成 */
    carMeshes.add(carBody)

    // new THREE.SpotLight(色, 光の強さ, 距離, 照射角, ボケ具合, 減衰率)
    /** ヘッドライトを追加 */
    const hedlight = () => {
      const hedlight = new THREE.SpotLight(
        0xffffff, // ヘッドライトの色を設定
        100, // 光のカンデラを設定
        0, // 光の届く距離を設定
        Math.PI / 12, // 照射角を設定
        2, // ボケ具合を設定
        1, // 減衰率を設定
      ) // ヘッドライトを作成
      hedlight.position.set(-1.91, -0.2, 0) // 車の前方に配置
      hedlight.power = 100 // 光の強さを設定
      hedlight.castShadow = true // ヘッドライトが影を落とすように設定
      hedlight.shadow.mapSize.width = 1024 // シャドウマップの幅
      hedlight.shadow.mapSize.height = 1024 // シャドウマップの高さ
      hedlight.target.position.set(-10, -1.3, 0) // ヘッドライトの照射先を設定
      return hedlight
    }
    const hedlight1 = hedlight() // ヘッドライトを生成
    hedlight1.position.z = 0.8 // ヘッドライトの位置を調整
    hedlight1.target.position.z = 0.8 // ヘッドライトの照射先の位置を調整
    carMeshes.add(hedlight1) // ヘッドライトを車のメッシュに追加
    carMeshes.add(hedlight1.target) // ヘッドライトの照射先を車のメッシュに追加

    const hedlight2 = hedlight() // ヘッドライトを生成
    hedlight2.position.z = -0.8 // ヘッドライトの位置を調整
    hedlight2.target.position.z = -0.8 // ヘッドライトの照射先の位置を調整
    carMeshes.add(hedlight2) // ヘッドライトを車のメッシュに追加
    carMeshes.add(hedlight2.target) // ヘッドライトの照射先を車のメッシュに追加

    carMeshes.add(new THREE.SpotLightHelper(hedlight1)) // ヘッドライトのヘルパーを追加
    carMeshes.add(new THREE.SpotLightHelper(hedlight2)) // ヘッドライトのヘルパーを追加

    /** 車体のボディー */
    const car = {
      name: 'car',
      vehicle: true,
      myVehicle: true, // この車は自分が操縦する車であることを示すフラグ
      controlable: true, // この車は制御可能であることを示す
      mass: 1000, // 車の質量を設定
      position: new CANNON.Vec3(0, 3, 5),
      shape: new CANNON.Box(new CANNON.Vec3(1.9, 0.6, 0.8)), // 車の形状を箱型に設定
      mesh: carMeshes, // 車のメッシュを設定
      wheelOptions: {
        radius: 0.2, // 車輪の半径を設定
        directionLocal: new CANNON.Vec3(0, -1, 0), // 車輪の方向を設定
        suspensionStiffness: 50, // サスペンションの剛性を設定
        suspensionRestLength: 0.5, // サスペンションの初期長さを設定
        frictionSlip: 5, // 摩擦係数を設定
        dampingRelaxation: 2.3, // ダンピングの緩和を設定
        dampingCompression: 4.4, // 跳ね返りの強さを設定
        maxSuspensionForce: 100000, // 最大サスペンション力を設定
        rollInfluence: 0.01, // 車輪の転がり影響を設定
        axleLocal: new CANNON.Vec3(0, 0, 1), // 車輪の軸を設定
        chassisConnectionPointLocal: new CANNON.Vec3(-1, 0, 1), // 車体との接続点を設定
        maxSuspensionTravel: 0.3, // 最大サスペンションの移動距離を設定
        customSlidingRotationalSpeed: -30, // 車輪の回転速度を設定
        useCustomSlidingRotationalSpeed: true, // カスタム回転速度を使用
      },
    }

    objectList.push(car)

    //objectListから生成した、bodyとmeshのリスト
    const bodyMeshList = []

    /**
     * 物体を追加する関数
     * @param {Array} objectList - 物体のリスト
     */
    const addObjects = (objectList) => {
      let i = 0
      for (const object of objectList) {
        /** CANNON.jsの物理ボディを作成 */
        const body = this.createBody(object)

        /** THREE.jsのメッシュを作成 */
        const mesh = object.mesh
        mesh.castShadow = true // メッシュが影を落とすように設定
        mesh.receiveShadow = true // メッシュが影を受け取るように設定
        mesh.position.copy(body.position)
        mesh.quaternion.copy(body.quaternion)

        /** ホイール情報リスト */
        const wheelInfoArray = []

        //車両オブジェクトの場合の設定
        if (object.vehicle) {
          const raycastVehicle = new CANNON.RaycastVehicle({
            chassisBody: body, // 車体の物理ボディを設定
          })
          object.wheelOptions.chassisConnectionPointLocal.set(-1.3, -0.1, 0.7) //車幅はここで設定する
          raycastVehicle.addWheel(object.wheelOptions)
          object.wheelOptions.chassisConnectionPointLocal.set(-1.3, -0.1, -0.7)
          raycastVehicle.addWheel(object.wheelOptions)
          object.wheelOptions.chassisConnectionPointLocal.set(1.3, -0.1, 0.7)
          raycastVehicle.addWheel(object.wheelOptions)
          object.wheelOptions.chassisConnectionPointLocal.set(1.3, -0.1, -0.7)
          raycastVehicle.addWheel(object.wheelOptions)

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
                0.3, // 車輪の幅
                20, // クオリティ
              ),
              new CANNON.Vec3(), // 車輪の位置を設定
              q, // 車輪の回転を設定
            )
            const wheelCylinderGeometry = new THREE.CylinderGeometry(
              wheel.radius,
              wheel.radius,
              0.3, // 車輪の幅
              32, // 何角形にするか？
            )
            wheelCylinderGeometry.rotateX(Math.PI / 2) // 車輪を横向きにするための回転
            const wheelMesh = new THREE.Mesh(
              wheelCylinderGeometry,
              new THREE.MeshStandardMaterial({
                color: 0x000000,
                roughness: 0.1, // 非光沢度を設定
                metalness: 0.1, // 金属感を設定
              }),
            )
            wheelMesh.castShadow = true // メッシュが影を落とすように設定
            wheelMesh.receiveShadow = true // メッシュが影を受け取るように設定
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

          if (object.myVehicle) {
            myVehicle = raycastVehicle // 自分が操縦する車両を設定
            myVehicle.addToWorld(world) // RaycastVehicleを物理世界に追加
            world.addEventListener('postStep', () => {
              // 車両の位置を更新
              const forward = new THREE.Vector3(
                this.cameraForward.x,
                this.cameraForward.y,
                this.cameraForward.z,
              ) // 前方方向
              forward.applyQuaternion(myVehicle.chassisBody.quaternion) // 車両のクォータニオンを適用
              this.cameraPos.x =
                myVehicle.chassisBody.position.x + forward.x * 12
              this.cameraPos.y =
                myVehicle.chassisBody.position.y + forward.y * 3
              this.cameraPos.z =
                myVehicle.chassisBody.position.z + forward.z * 12
              // カメラの位置を更新
              this.cameraLookAt.x =
                myVehicle.chassisBody.position.x + forward.x * -30
              this.cameraLookAt.y =
                myVehicle.chassisBody.position.y + forward.y * -1
              this.cameraLookAt.z =
                myVehicle.chassisBody.position.z + forward.z * -30
            })
          } else {
            raycastVehicle.addToWorld(world) // RaycastVehicleを物理世界に追加
          }
        }

        let isStatic = object.mass === 0 // 質量が0なら静的な物体と判断
        if (object.isStatic) {
          isStatic = object.isStatic // isStaticフラグが設定されている場合はそれを使用
        }

        bodyMeshList.push({
          body: body,
          mesh: mesh,
          isVehicle: object.vehicle,
          isStatic: isStatic, // 静的な物体かどうかを示すフラグ
          wheelInfoArray: object.vehicle ? wheelInfoArray : null,
        })

        world.addBody(bodyMeshList[i].body)
        scene.add(bodyMeshList[i].mesh)

        i++
      }
    }

    addObjects(objectList)

    /**
     * 物理ボディの位置と回転をTHREE.jsのメッシュにコピーする関数
     * @returns {void}
     */
    const copy = () => {
      // CANNON.jsの物理ボディの位置と回転をTHREE.jsのメッシュにコピー
      for (const bodyMesh of bodyMeshList) {
        if (bodyMesh.isStatic) continue // 静的な物体はスキップ
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

    /** 太陽光 */
    const directionalLight = new THREE.DirectionalLight(0xffffff, 2)
    directionalLight.position.set(20, 100, 50)
    directionalLight.castShadow = true // 影を有効にする
    directionalLight.shadow.mapSize.width = 1024 // シャドウマップの幅を設定
    directionalLight.shadow.mapSize.height = 1024 // シャドウマップの高さを設定
    directionalLight.shadow.camera.right = 10
    directionalLight.shadow.camera.left = -10
    directionalLight.shadow.camera.top = 10
    directionalLight.shadow.camera.bottom = -10
    scene.add(directionalLight)

    this.initThree(scene, camera, renderer, copy, world, cubeCamera)

    document.onkeydown = (e) => {
      controlVehicle(e, myVehicle)
    }
    document.onkeyup = (e) => {
      controlVehicle(e, myVehicle)
    }

    const controlVehicle = (e, vehicle) => {
      const maxSteerVal = 0.8
      const maxForce = 2000
      const brakeForce = 1000000

      const keyup = e.type === 'keyup'
      if (!keyup && e.type !== 'keydown') return

      vehicle.setBrake(0, 0)
      vehicle.setBrake(0, 1)
      vehicle.setBrake(0, 2)
      vehicle.setBrake(0, 3)

      switch (e.keyCode) {
        case key.up: // forward
        case key.w: // w
          vehicle.applyEngineForce(keyup ? 0 : -maxForce, 2)
          vehicle.applyEngineForce(keyup ? 0 : -maxForce, 3)
          break

        case key.down: // backward
        case key.s: // s
          vehicle.applyEngineForce(keyup ? 0 : maxForce, 2)
          vehicle.applyEngineForce(keyup ? 0 : maxForce, 3)
          break

        case key.b: // b
          //サイドブレーキ仕様で、後輪のみ
          //vehicle.setBrake(brakeForce, 0)
          //vehicle.setBrake(brakeForce, 1)
          vehicle.setBrake(brakeForce, 2)
          vehicle.setBrake(brakeForce, 3)
          break

        case key.right: // right
        case key.d: // d
          vehicle.setSteeringValue(keyup ? 0 : -maxSteerVal, 0)
          vehicle.setSteeringValue(keyup ? 0 : -maxSteerVal, 1)
          break

        case key.left: // left
        case key.a: // a
          vehicle.setSteeringValue(keyup ? 0 : maxSteerVal, 0)
          vehicle.setSteeringValue(keyup ? 0 : maxSteerVal, 1)
          break

        case key.q: // q
          this.cameraForward.x = keyup ? 1 : -1
          this.cameraForward.y = keyup ? 1 : 0.3
          this.cameraForward.z = keyup ? 0 : 0.5
          break

        case key.e: // e
          this.cameraForward.x = keyup ? 1 : -1
          this.cameraForward.y = keyup ? 1 : 0.3
          this.cameraForward.z = keyup ? 0 : -0.5
      }
    }
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
    initThree(scene, camera, renderer, copy, world, cubeCamera) {
      /*
      new RGBELoader().load(
        '/cloud_1k.hdr', // HDRIのパス
        (texture) => {
          texture.mapping = THREE.EquirectangularReflectionMapping // HDRIを環境マッピングに設定
          scene.environment = texture // シーンの環境マッピングに設定
        },
      )
      new RGBELoader().load(
        '/cloud_4k.hdr', // HDRIのパス
        (texture) => {
          texture.mapping = THREE.EquirectangularReflectionMapping // HDRIを環境マッピングに設定
          scene.background = texture // シーンの背景に設定
        },
      )
        */

      //DOMサイズ取得
      const clientWidth =
        window.innerWidth ||
        document.documentElement.clientWidth ||
        document.body.clientWidth
      const clientHeight =
        window.innerHeight ||
        document.documentElement.clientHeight ||
        document.body.clientHeight

      //カメラ設定
      camera.aspect = clientWidth / clientHeight
      camera.updateProjectionMatrix()
      camera.position.set(5, 3, 20)
      camera.lookAt(0, 4.5, 0) // カメラの注視点を設定

      //レンダラー設定
      renderer.setSize(clientWidth, clientHeight)
      renderer.setPixelRatio(clientWidth / clientHeight) // ピクセル比を設定
      this.$refs.container.appendChild(renderer.domElement)

      const animate = () => {
        const frame = () => {
          copy()
          cubeCamera.update(renderer, scene) // 環境マッピングの更新
          camera.position.set(
            this.cameraPos.x,
            this.cameraPos.y,
            this.cameraPos.z,
          )
          camera.lookAt(
            this.cameraLookAt.x,
            this.cameraLookAt.y,
            this.cameraLookAt.z,
          )
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
