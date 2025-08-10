<div style="position: relative; width: 100%; height: 100vh;">
  <div bind:this={container} style="width: 100%; height: 100%;"></div>
  <div style="position: absolute; top: 20px; left: 20px; z-index: 1000;">
    <button 
      onclick={applyRandomImpulse} 
      style="padding: 10px 20px; font-size: 16px; background: #007acc; color: white; border: none; border-radius: 5px; cursor: pointer;">
      Add Impulse
    </button>
    <button 
      onclick={resetRagdoll} 
      style="padding: 10px 20px; font-size: 16px; background: #cc4400; color: white; border: none; border-radius: 5px; cursor: pointer; margin-left: 10px;">
      Reset
    </button>
    <button 
      onclick={toggleDebugMode} 
      style="padding: 10px 20px; font-size: 16px; background: {debugMode ? '#ff6600' : '#666666'}; color: white; border: none; border-radius: 5px; cursor: pointer; margin-left: 10px;">
      Debug {debugMode ? 'ON' : 'OFF'}
    </button>
  </div>
</div>

<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import RAPIER from '@dimforge/rapier3d-compat';

  let container = $state();
  let scene, camera, renderer, world, controls;
  let rigidBodies = [];
  let meshes = [];
  let physicsBones = {}; // Map of bone names to physics-driven bone copies
  let debugMode = $state(false);
  let debugHelpers = [];
  
  // Collision groups
  const GROUND_GROUP = 1;
  const RAGDOLL_GROUP = 2;
  const CHARACTER_GROUP = 4;

  onMount(async () => {
    await RAPIER.init();
    initScene();
    initPhysics();
    loadModel();
    animate();
  });

  function initScene() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x222222);

    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 10, 20);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    container.appendChild(renderer.domElement);

    // Add orbit controls
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.target.set(0, 2, 0);
    controls.minDistance = 1;
    controls.maxDistance = 30;
    controls.maxPolarAngle = Math.PI * 0.9; // Prevent going under ground
    controls.minPolarAngle = Math.PI * 0.1; // Prevent going too high

    const ambientLight = new THREE.AmbientLight(0x404040, 0.6);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(-1, 1, 1);
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize.width = 2048;
    directionalLight.shadow.mapSize.height = 2048;
    scene.add(directionalLight);

    const groundGeometry = new THREE.PlaneGeometry(20, 20);
    const groundMaterial = new THREE.MeshLambertMaterial({ color: 0x888888 });
    const ground = new THREE.Mesh(groundGeometry, groundMaterial);
    ground.rotation.x = -Math.PI / 2;
    ground.receiveShadow = true;
    scene.add(ground);

    window.addEventListener('resize', onWindowResize);
  }

  function initPhysics() {
    const gravity = new RAPIER.Vector3(0.0, -9.81, 0.0);
    world = new RAPIER.World(gravity);

    // Ground collider with proper collision filtering
    const groundBodyDesc = RAPIER.RigidBodyDesc.fixed();
    const groundBody = world.createRigidBody(groundBodyDesc);
    const groundColliderDesc = RAPIER.ColliderDesc.cuboid(10, 0.1, 10);
    const groundCollider = world.createCollider(groundColliderDesc, groundBody);
    
    console.log('Ground collider created at position:', groundBody.translation());
    console.log('Ground collider size: 20x0.2x20 units');
    
    // Make sure ground is at Y=0 and visible
    const groundMesh = scene.getObjectByName('ground') || scene.children.find(child => 
      child.geometry && child.geometry.type === 'PlaneGeometry'
    );
    if (groundMesh) {
      console.log('Visual ground mesh position:', groundMesh.position);
    }
  }

  async function loadModel() {
    const loader = new GLTFLoader();
    
    try {
      const gltf = await new Promise((resolve, reject) => {
        loader.load('/model.glb', resolve, undefined, reject);
      });

      const model = gltf.scene;
      model.position.set(0, 8, 0); // Raised higher to avoid floor clipping
      
      // Make sure all meshes are visible and have shadows
      model.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true;
          child.receiveShadow = true;
          child.visible = true;
        }
      });
      
      scene.add(model);
      createRagdoll(model);
    } catch (error) {
      console.error('Error loading model:', error);
    }
  }

  function createRagdoll(model) {
    const colliderObjects = [];
    let skinnedMesh = null;
    let originalBones = [];
    
    // Reset physics bones map
    physicsBones = {};
    
    // Find collision objects and skinned mesh
    model.traverse((child) => {
      if (child.isMesh) {
        if (child.name && child.name.startsWith('Col_')) {
          // This is a collision object
          colliderObjects.push(child);
          child.visible = false; // Hide collision meshes
          console.log('Found collision object:', child.name);
        } else if (child.isSkinnedMesh) {
          // This is the main skinned mesh
          skinnedMesh = child;
          originalBones = child.skeleton.bones;
          child.castShadow = true;
          child.receiveShadow = true;
          child.visible = true;
          console.log('Found skinned mesh with', originalBones.length, 'bones');
        }
      }
    });

    if (!skinnedMesh) {
      console.error('No skinned mesh found!');
      return;
    }

    console.log('=== GLB STRUCTURE DEBUG ===');
    console.log('Available collision objects:', colliderObjects.map(c => c.name));
    console.log('Available bones:', originalBones.map(b => b.name));
    
    // Special check for leg-related items
    const legColliders = colliderObjects.filter(c => c.name.toLowerCase().includes('leg'));
    const legBones = originalBones.filter(b => b.name.toLowerCase().includes('leg'));
    console.log('LEG COLLIDERS FOUND:', legColliders.map(c => c.name));
    console.log('LEG BONES FOUND:', legBones.map(b => b.name));
    
    if (legColliders.length === 0) {
      console.error('❌ NO LEG COLLISION OBJECTS FOUND! Check GLB file.');
      alert('NO LEG COLLISION OBJECTS FOUND! Check browser console for details.');
    }
    if (legBones.length === 0) {
      console.error('❌ NO LEG BONES FOUND! Check bone naming.');
      alert('NO LEG BONES FOUND! Check browser console for details.');
    }
    
    // Also check for common leg naming variations
    const legVariations = ['thigh', 'shin', 'calf', 'femur', 'tibia'];
    legVariations.forEach(variation => {
      const found = colliderObjects.filter(c => c.name.toLowerCase().includes(variation));
      if (found.length > 0) {
        console.log(`Found collision objects with "${variation}":`, found.map(c => c.name));
      }
    });
    
    // Debug collision object details
    colliderObjects.forEach(colObj => {
      const pos = new THREE.Vector3();
      const scale = new THREE.Vector3(); 
      colObj.getWorldPosition(pos);
      colObj.getWorldScale(scale);
      console.log(`Collision object ${colObj.name}:`, {
        position: { x: pos.x.toFixed(2), y: pos.y.toFixed(2), z: pos.z.toFixed(2) },
        scale: { x: scale.x.toFixed(2), y: scale.y.toFixed(2), z: scale.z.toFixed(2) }
      });
    });
    
    // Debug bone hierarchy
    console.log('Bone hierarchy:');
    originalBones.forEach(bone => {
      const pos = new THREE.Vector3();
      bone.getWorldPosition(pos);
      console.log(`  ${bone.name} (parent: ${bone.parent ? bone.parent.name : 'ROOT'}) pos: ${pos.x.toFixed(2)}, ${pos.y.toFixed(2)}, ${pos.z.toFixed(2)}`);
    });

    // CORRECT APPROACH: Only create physics bodies for bones that have corresponding collision objects
    colliderObjects.forEach((colObj) => {
      console.log(`\n=== Processing collision object: ${colObj.name} ===`);
      
      // Find the bone that corresponds to this collision object
      const boneName = colObj.name.replace('Col_', '');
      console.log(`Looking for bone matching: "${boneName}"`);
      
      let correspondingBone = originalBones.find(bone => bone.name === boneName);
      console.log(`Exact match result:`, correspondingBone ? correspondingBone.name : 'NOT FOUND');
      
      // If exact match not found, try partial matching with detailed logging
      if (!correspondingBone) {
        console.log('Trying partial matching...');
        const lowerColName = boneName.toLowerCase();
        
        originalBones.forEach(bone => {
          const lowerBoneName = bone.name.toLowerCase();
          const matches = lowerBoneName.includes(lowerColName) || 
                         lowerColName.includes(lowerBoneName) ||
                         (lowerColName.includes('lowerleg') && lowerBoneName.includes('leg')) ||
                         (lowerColName.includes('upperleg') && lowerBoneName.includes('leg')) ||
                         (lowerColName.includes('lowerarm') && lowerBoneName.includes('arm')) ||
                         (lowerColName.includes('upperarm') && lowerBoneName.includes('arm'));
          
          if (matches) {
            console.log(`  Potential match: "${bone.name}" (${lowerBoneName}) matches "${boneName}" (${lowerColName})`);
            if (!correspondingBone) correspondingBone = bone;
          }
        });
      }
      
      if (!correspondingBone) {
        console.error(`❌ NO BONE FOUND for collision object: ${colObj.name} (looking for: ${boneName})`);
        console.log('Available bone names:', originalBones.map(b => b.name));
        return;
      }
      
      console.log(`✅ Matched collision object ${colObj.name} to bone ${correspondingBone.name}`);
      
      // Get collision object geometry for accurate physics shape
      const geometry = colObj.geometry;
      if (!geometry) {
        console.warn(`No geometry for collision object: ${colObj.name}`);
        return;
      }
      
      geometry.computeBoundingBox();
      const size = geometry.boundingBox.getSize(new THREE.Vector3());
      
      // Get world transforms
      const colWorldPos = new THREE.Vector3();
      const colWorldScale = new THREE.Vector3();
      const colWorldQuat = new THREE.Quaternion();
      colObj.getWorldPosition(colWorldPos);
      colObj.getWorldScale(colWorldScale);
      colObj.getWorldQuaternion(colWorldQuat);
      
      // Use collision object position and size directly - the GLB has them positioned correctly!
      const scaledSize = size.clone().multiply(colWorldScale);
      
      console.log(`Collision object ${colObj.name} positioned at:`, {
        x: colWorldPos.x.toFixed(2), 
        y: colWorldPos.y.toFixed(2), 
        z: colWorldPos.z.toFixed(2)
      });
      
      const bodyDesc = RAPIER.RigidBodyDesc.dynamic()
        .setTranslation(colWorldPos.x, colWorldPos.y, colWorldPos.z) // Use collision object position
        .setRotation(colWorldQuat) // Use collision object rotation
        .setLinearDamping(0.95)
        .setAngularDamping(0.95);
      
      const rigidBody = world.createRigidBody(bodyDesc);
      
      // Debug collider size for legs specifically
      if (colObj.name.toLowerCase().includes('leg')) {
        console.log(`Creating leg collider for ${colObj.name}:`, {
          originalSize: size,
          worldScale: colWorldScale,
          scaledSize: scaledSize,
          halfSizes: { 
            x: Math.max(scaledSize.x * 0.5, 0.03),
            y: Math.max(scaledSize.y * 0.5, 0.03), 
            z: Math.max(scaledSize.z * 0.5, 0.03)
          }
        });
      }
      
      const colliderDesc = RAPIER.ColliderDesc.cuboid(
        Math.max(scaledSize.x * 0.5, 0.03),
        Math.max(scaledSize.y * 0.5, 0.03),
        Math.max(scaledSize.z * 0.5, 0.03)
      );
      
      const collider = world.createCollider(colliderDesc, rigidBody);
      
      // Ensure all colliders can hit the ground (no collision filtering)
      // collider.setCollisionGroups(0xFFFF); // Collide with everything
      // collider.setSolverGroups(0xFFFF);    // Solve with everything
      
      // Make sure leg colliders can collide with ground
      if (colObj.name.toLowerCase().includes('leg')) {
        console.log(`Leg collider created at position:`, {
          x: colWorldPos.x.toFixed(2),
          y: colWorldPos.y.toFixed(2), 
          z: colWorldPos.z.toFixed(2)
        });
        
        // Extra verification for leg collision
        console.log(`Leg ${colObj.name} physics body handle:`, rigidBody.handle);
        console.log(`Leg ${colObj.name} collider handle:`, collider.handle);
      }
      
      // Calculate offset from collision object to bone for proper bone updates
      const boneWorldPos = new THREE.Vector3();
      const boneWorldQuat = new THREE.Quaternion();
      correspondingBone.updateMatrixWorld(true);
      correspondingBone.getWorldPosition(boneWorldPos);
      correspondingBone.getWorldQuaternion(boneWorldQuat);
      
      // Offset from collision object position to bone position
      const collisionToBoneOffset = boneWorldPos.clone().sub(colWorldPos);
      
      // Calculate initial rotation difference between collision object and bone
      const collisionToBoneRotation = boneWorldQuat.clone().premultiply(colWorldQuat.clone().invert());
      
      console.log(`Offset from collision ${colObj.name} to bone ${correspondingBone.name}:`, {
        pos: { x: collisionToBoneOffset.x.toFixed(2), y: collisionToBoneOffset.y.toFixed(2), z: collisionToBoneOffset.z.toFixed(2) },
        rot: { x: collisionToBoneRotation.x.toFixed(2), y: collisionToBoneRotation.y.toFixed(2), z: collisionToBoneRotation.z.toFixed(2), w: collisionToBoneRotation.w.toFixed(2) }
      });
      
      // Store physics bone data - only for bones with collision objects
      physicsBones[correspondingBone.name] = {
        originalBone: correspondingBone,
        physicsBody: rigidBody,
        colliderObject: colObj,
        collisionToBoneOffset: collisionToBoneOffset,
        collisionToBoneRotation: collisionToBoneRotation,
        initialLocalPosition: correspondingBone.position.clone(),
        initialLocalQuaternion: correspondingBone.quaternion.clone(),
        initialLocalScale: correspondingBone.scale.clone()
      };
      
      rigidBodies.push(rigidBody);
      meshes.push(correspondingBone);
      
      console.log(`Created physics body for bone ${correspondingBone.name} using collision object ${colObj.name}`);
    });
    
    // Store reference to skinned mesh for updates
    if (skinnedMesh && !meshes.includes(skinnedMesh)) {
      meshes.push(skinnedMesh);
    }
    
    console.log(`Created ${rigidBodies.length} physics bodies for ragdoll bones`);
    console.log('Physics bones created:', Object.keys(physicsBones));
    
    // Log bone hierarchy for debugging
    console.log('Bone hierarchy:');
    originalBones.forEach(bone => {
      console.log(`  ${bone.name} -> ${bone.parent ? bone.parent.name : 'ROOT'}`);
    });
    
    // Add very small initial impulse - joints are now stiffer
    rigidBodies.forEach((body, index) => {
      body.applyImpulse(new RAPIER.Vector3(
        (Math.random() - 0.5) * 0.05,
        -0.2,
        (Math.random() - 0.5) * 0.05
      ), true);
    });
    
    // Create joints only between physics-driven bones (simplified approach)
    const boneConnections = [];
    const physicsBonesArray = Object.values(physicsBones);
    
    // Create joints based on proximity and naming patterns between physics bones
    for (let i = 0; i < physicsBonesArray.length; i++) {
      for (let j = i + 1; j < physicsBonesArray.length; j++) {
        const boneData1 = physicsBonesArray[i];
        const boneData2 = physicsBonesArray[j];
        const bone1 = boneData1.originalBone;
        const bone2 = boneData2.originalBone;
        
        // Check if these bones should be connected
        let shouldConnect = false;
        let jointType = 'default';
        
        const name1 = bone1.name.toLowerCase();
        const name2 = bone2.name.toLowerCase();
        
        // Check for direct parent-child relationship
        if (bone1.parent === bone2 || bone2.parent === bone1) {
          shouldConnect = true;
          
          // Determine joint type
          if ((name1.includes('head') && name2.includes('torso')) || (name2.includes('head') && name1.includes('torso'))) {
            jointType = 'neck';
          } else if ((name1.includes('upperarm') && name2.includes('lowerarm')) || (name2.includes('upperarm') && name1.includes('lowerarm'))) {
            jointType = 'elbow';
          } else if ((name1.includes('upperleg') && name2.includes('lowerleg')) || (name2.includes('upperleg') && name1.includes('lowerleg'))) {
            jointType = 'knee';
          } else if ((name1.includes('torso') && name2.includes('upperarm')) || (name2.includes('torso') && name1.includes('upperarm'))) {
            jointType = 'shoulder';
          } else if ((name1.includes('torso') && name2.includes('upperleg')) || (name2.includes('torso') && name1.includes('upperleg'))) {
            jointType = 'hip';
          }
        }
        
        if (shouldConnect) {
          boneConnections.push({
            boneData1: boneData1,
            boneData2: boneData2,
            jointType: jointType
          });
          
          console.log(`Will create joint: ${bone1.name} <-> ${bone2.name} (${jointType})`);
        }
      }
    }
    
    // Create joints for the connections we found
    boneConnections.forEach(connection => {
      const { boneData1, boneData2, jointType } = connection;
      const body1 = boneData1.physicsBody;
      const body2 = boneData2.physicsBody;
      const bone1 = boneData1.originalBone;
      const bone2 = boneData2.originalBone;
      
      if (body1 && body2) {
        try {
                // Create a ball joint (spherical) that allows rotation but maintains connection
                const pos1 = body1.translation();
                const pos2 = body2.translation();
                
                // Calculate midpoint for joint anchor
                const midX = (pos1.x + pos2.x) * 0.5;
                const midY = (pos1.y + pos2.y) * 0.5;
                const midZ = (pos1.z + pos2.z) * 0.5;
                
                // Set joint limits based on joint type
                let swingLimit1, swingLimit2, twistLimit;
                
                switch (jointType) {
                  case 'elbow':
                    swingLimit1 = Math.PI * 0.5; // 90 degrees - more realistic elbow bend
                    swingLimit2 = Math.PI * 0.02; // 3.6 degrees - very minimal side movement
                    twistLimit = Math.PI * 0.02;  // 3.6 degrees - very minimal twist
                    break;
                  case 'knee':
                    swingLimit1 = Math.PI * 0.7; // 126 degrees - knee can bend a lot
                    swingLimit2 = Math.PI * 0.02; // 3.6 degrees - very limited side movement
                    twistLimit = Math.PI * 0.02;  // 3.6 degrees - very limited twist
                    break;
                  case 'shoulder':
                    swingLimit1 = Math.PI * 0.5; // 90 degrees - good shoulder movement
                    swingLimit2 = Math.PI * 0.4; // 72 degrees - shoulder can move around
                    twistLimit = Math.PI * 0.25; // 45 degrees - some twist
                    break;
                  case 'hip':
                    swingLimit1 = Math.PI * 0.35; // 63 degrees - hip movement
                    swingLimit2 = Math.PI * 0.25; // 45 degrees - hip side movement
                    twistLimit = Math.PI * 0.15;  // 27 degrees - hip twist
                    break;
                  case 'neck':
                    swingLimit1 = Math.PI * 0.2; // 36 degrees - reasonable neck forward/back
                    swingLimit2 = Math.PI * 0.15; // 27 degrees - reasonable neck side movement  
                    twistLimit = Math.PI * 0.25;   // 45 degrees - reasonable head turning
                    break;
                  case 'spine':
                    swingLimit1 = Math.PI * 0.1; // 18 degrees - limited spine forward/back
                    swingLimit2 = Math.PI * 0.08; // 14 degrees - limited spine side movement
                    twistLimit = Math.PI * 0.05; // 9 degrees - very limited spine twist
                    break;
                  default:
                    swingLimit1 = Math.PI * 0.15; // Reduced default limits
                    swingLimit2 = Math.PI * 0.15;
                    twistLimit = Math.PI * 0.1;
                }
                
                // Set stiffness and damping based on joint type
                let stiffness, damping;
                
                switch (jointType) {
                  case 'elbow':
                  case 'knee':
                    stiffness = 100000; // Extremely stiff hinge joints to stay close to mesh
                    damping = 4000;     // Very high damping to prevent oscillation
                    break;
                  case 'shoulder':
                  case 'hip':
                    stiffness = 30000;  // Moderate stiffness for ball joints
                    damping = 1500;     // Good damping
                    break;
                  case 'neck':
                    stiffness = 80000;  // Moderate stiffness for natural head movement
                    damping = 3000;     // Good damping without being too rigid
                    break;
                  case 'spine':
                    stiffness = 200000; // Very stiff spine for stability
                    damping = 8000;     // High damping to prevent spine wiggling
                    break;
                  default:
                    stiffness = 50000;  // Increased default stiffness
                    damping = 2000;
                }

                const joint = {
                  type: 'ball',
                  body1: body1.handle,
                  body2: body2.handle,
                  anchor1: { 
                    x: midX - pos1.x, 
                    y: midY - pos1.y, 
                    z: midZ - pos1.z 
                  },
                  anchor2: { 
                    x: midX - pos2.x, 
                    y: midY - pos2.y, 
                    z: midZ - pos2.z 
                  },
                  limits: {
                    swing1: swingLimit1,
                    swing2: swingLimit2,
                    twist: twistLimit
                  },
                  stiffness: stiffness,
                  damping: damping
                };
                
                world.createImpulseJoint(joint, true);
                console.log(`Created ${jointType} joint between ${bone1.name} and ${bone2.name} (swing1: ${(swingLimit1 * 180 / Math.PI).toFixed(0)}°, swing2: ${(swingLimit2 * 180 / Math.PI).toFixed(0)}°)`);
          } catch (error) {
            console.log(`Failed to create joint between ${bone1.name} and ${bone2.name}:`, error);
            
            // Fallback: create simple distance constraint
            body1.userData = body1.userData || {};
            body1.userData.constraints = body1.userData.constraints || [];
            
            const pos1 = body1.translation();
            const pos2 = body2.translation();
            const distance = Math.sqrt(
              Math.pow(pos1.x - pos2.x, 2) + 
              Math.pow(pos1.y - pos2.y, 2) + 
              Math.pow(pos1.z - pos2.z, 2)
            );
            
            // Extra strong constraint for head connections and critical joints
            let constraintStrength;
            if (jointType === 'neck') {
              constraintStrength = 0.995; // Ultra strong neck connection - nearly rigid
            } else if (jointType === 'elbow' || jointType === 'knee') {
              constraintStrength = 0.95; // Very strong limb connections
            } else {
              constraintStrength = 0.9; // Strong default
            }
            
            body1.userData.constraints.push({
              targetBody: body2,
              targetDistance: distance,
              strength: constraintStrength
            });
            
            console.log(`Created fallback constraint between ${bone1.name} and ${bone2.name}`);
          }
        }
    });
    
    console.log(`Created joints based on bone hierarchy: ${boneConnections.length} connections`);
  }

  function animate() {
    requestAnimationFrame(animate);
    
    world.step();
    
    // Handle any fallback distance constraints
    rigidBodies.forEach((body) => {
      if (body.userData && body.userData.constraints) {
        body.userData.constraints.forEach((constraint) => {
          const pos1 = body.translation();
          const pos2 = constraint.targetBody.translation();
          
          const currentDistance = Math.sqrt(
            Math.pow(pos1.x - pos2.x, 2) + 
            Math.pow(pos1.y - pos2.y, 2) + 
            Math.pow(pos1.z - pos2.z, 2)
          );
          
          const targetDistance = constraint.targetDistance;
          // Ultra tight stretch limits based on constraint strength
          let maxStretch;
          if (constraint.strength > 0.99) {
            maxStretch = targetDistance * 1.01; // Only 1% stretch for ultra strong neck
          } else if (constraint.strength > 0.94) {
            maxStretch = targetDistance * 1.05; // 5% stretch for elbows/knees  
          } else {
            maxStretch = targetDistance * 1.08; // 8% stretch for others
          }
          
          if (currentDistance > maxStretch) {
            // Gently pull bodies back together
            const direction = {
              x: (pos1.x - pos2.x) / currentDistance,
              y: (pos1.y - pos2.y) / currentDistance,
              z: (pos1.z - pos2.z) / currentDistance
            };
            
            const correction = (currentDistance - maxStretch) * constraint.strength * 0.1;
            
            body.setTranslation({
              x: pos1.x - direction.x * correction,
              y: pos1.y - direction.y * correction,
              z: pos1.z - direction.z * correction
            }, true);
          }
        });
      }
    });
    
    // Update ONLY physics-driven bones from physics bodies
    // Let the skeleton naturally handle non-physics bones
    Object.values(physicsBones).forEach(boneData => {
      const { originalBone, physicsBody, collisionToBoneOffset, collisionToBoneRotation } = boneData;
      
      if (physicsBody && originalBone) {
        const physicsPos = physicsBody.translation();
        const physicsRot = physicsBody.rotation();
        
        // Get physics world transform (this is at collision object position/rotation)
        const physicsWorldPos = new THREE.Vector3(physicsPos.x, physicsPos.y, physicsPos.z);
        const physicsWorldQuat = new THREE.Quaternion(physicsRot.x, physicsRot.y, physicsRot.z, physicsRot.w);
        
        // Calculate where the bone should be by applying the position offset
        const boneWorldPos = physicsWorldPos.clone();
        if (collisionToBoneOffset) {
          // Apply the offset, rotated by current physics rotation
          const rotatedOffset = collisionToBoneOffset.clone().applyQuaternion(physicsWorldQuat);
          boneWorldPos.add(rotatedOffset);
        }
        
        // Calculate what the bone rotation should be by applying the rotation offset
        const boneWorldQuat = physicsWorldQuat.clone();
        if (collisionToBoneRotation) {
          // Apply the initial rotation difference
          boneWorldQuat.multiply(collisionToBoneRotation);
        }
        
        // Convert to local space if bone has parent
        if (originalBone.parent) {
          originalBone.parent.updateMatrixWorld(true);
          const parentInverseMatrix = new THREE.Matrix4().copy(originalBone.parent.matrixWorld).invert();
          
          // Convert bone world position to local space
          const localPos = boneWorldPos.clone().applyMatrix4(parentInverseMatrix);
          
          // Convert bone world rotation to local space
          const parentWorldQuat = new THREE.Quaternion();
          originalBone.parent.getWorldQuaternion(parentWorldQuat);
          const localQuat = boneWorldQuat.clone().premultiply(parentWorldQuat.clone().invert());
          
          // Apply local transforms
          originalBone.position.copy(localPos);
          originalBone.quaternion.copy(localQuat);
        } else {
          // Root bone - apply world transforms directly
          originalBone.position.copy(boneWorldPos);
          originalBone.quaternion.copy(boneWorldQuat);
        }
        
        // Preserve original scale
        originalBone.scale.copy(boneData.initialLocalScale);
        
        // Update matrices
        originalBone.updateMatrix();
        originalBone.updateMatrixWorld(true);
        
        // Debug: Check if legs are going through floor
        if (originalBone.name.toLowerCase().includes('leg') && boneWorldPos.y < 0.2) {
          console.warn(`⚠️ ${originalBone.name} is near/below ground! Y position: ${boneWorldPos.y.toFixed(2)}`);
          console.log(`Physics body Y: ${physicsPos.y.toFixed(2)}, Bone world Y: ${boneWorldPos.y.toFixed(2)}`);
        }
      }
    });
    
    // The skeleton will naturally handle all non-physics bones based on their relationships
    // to the physics-driven bones - no manual processing needed!
    
    // Update the skinned mesh
    scene.traverse((child) => {
      if (child.isSkinnedMesh && child.skeleton) {
        child.skeleton.update();
      }
    });
    
    // Update debug visuals
    updateDebugVisuals();
    
    // Update orbit controls
    controls.update();
    
    renderer.render(scene, camera);
  }

  function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }

  function applyRandomImpulse() {
    if (rigidBodies.length === 0) return;
    
    // Apply impulse to random body parts
    rigidBodies.forEach((body) => {
      const randomStrength = (Math.random() - 0.5) * 2; // -1 to 1
      body.applyImpulse(new RAPIER.Vector3(
        randomStrength,
        Math.abs(randomStrength) * 0.5, // Some upward force
        (Math.random() - 0.5) * 2
      ), true);
    });
    
    console.log('Applied random impulse to ragdoll');
  }

  function resetRagdoll() {
    if (rigidBodies.length === 0) return;
    
    // Reset all bodies to original positions with zero velocity
    Object.values(physicsBones).forEach(boneData => {
      const { physicsBody, originalBone } = boneData;
      
      if (physicsBody && originalBone) {
        // Get original position
        const originalPos = new THREE.Vector3();
        originalBone.getWorldPosition(originalPos);
        
        // Reset physics body
        physicsBody.setTranslation(originalPos, true);
        physicsBody.setRotation(new RAPIER.Quaternion(0, 0, 0, 1), true);
        physicsBody.setLinvel(new RAPIER.Vector3(0, 0, 0), true);
        physicsBody.setAngvel(new RAPIER.Vector3(0, 0, 0), true);
      }
    });
    
    console.log('Reset ragdoll to original position');
  }

  function toggleDebugMode() {
    debugMode = !debugMode;
    
    if (debugMode) {
      createDebugVisuals();
    } else {
      clearDebugVisuals();
    }
  }

  function createDebugVisuals() {
    clearDebugVisuals();
    
    Object.values(physicsBones).forEach(boneData => {
      const { physicsBody, originalBone } = boneData;
      
      if (physicsBody && originalBone) {
        // Create a wireframe sphere to show physics body position (RED)
        const physicsGeometry = new THREE.SphereGeometry(0.1, 8, 8);
        const physicsMaterial = new THREE.MeshBasicMaterial({ 
          color: 0xff0000, 
          wireframe: true 
        });
        const physicsSphere = new THREE.Mesh(physicsGeometry, physicsMaterial);
        
        // Create a smaller solid sphere to show bone position (GREEN)
        const boneGeometry = new THREE.SphereGeometry(0.05, 6, 6);
        const boneMaterial = new THREE.MeshBasicMaterial({ 
          color: 0x00ff00
        });
        const boneSphere = new THREE.Mesh(boneGeometry, boneMaterial);
        
        // Create axis helpers
        const physicsAxis = new THREE.AxesHelper(0.2);
        const boneAxis = new THREE.AxesHelper(0.15);
        
        physicsSphere.add(physicsAxis);
        boneSphere.add(boneAxis);
        
        scene.add(physicsSphere);
        scene.add(boneSphere);
        
        debugHelpers.push(physicsSphere);
        debugHelpers.push(boneSphere);
        
        // Store references for updates
        physicsSphere.userData = { 
          type: 'physics',
          physicsBody: physicsBody, 
          originalBone: originalBone 
        };
        boneSphere.userData = { 
          type: 'bone',
          physicsBody: physicsBody, 
          originalBone: originalBone 
        };
      }
    });
    
    console.log(`Created ${debugHelpers.length} debug helpers (red=physics, green=bone)`);
  }

  function clearDebugVisuals() {
    debugHelpers.forEach(helper => {
      scene.remove(helper);
    });
    debugHelpers = [];
  }

  function updateDebugVisuals() {
    if (!debugMode) return;
    
    debugHelpers.forEach(helper => {
      const { type, physicsBody, originalBone } = helper.userData;
      
      if (type === 'physics' && physicsBody) {
        // Show physics body position and rotation
        const pos = physicsBody.translation();
        const rot = physicsBody.rotation();
        
        helper.position.set(pos.x, pos.y, pos.z);
        helper.quaternion.set(rot.x, rot.y, rot.z, rot.w);
      } else if (type === 'bone' && originalBone) {
        // Show actual bone position and rotation
        const boneWorldPos = new THREE.Vector3();
        const boneWorldQuat = new THREE.Quaternion();
        
        originalBone.getWorldPosition(boneWorldPos);
        originalBone.getWorldQuaternion(boneWorldQuat);
        
        helper.position.copy(boneWorldPos);
        helper.quaternion.copy(boneWorldQuat);
      }
    });
  }
</script>