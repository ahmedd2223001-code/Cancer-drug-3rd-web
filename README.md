import React, { useState, useEffect, useRef, useMemo, Suspense } from 'react';
import { createRoot } from 'react-dom/client';
import { Canvas, useFrame, useThree } from '@react-three/fiber';
import { OrbitControls, PerspectiveCamera, Environment, ContactShadows, Text, Float, RoundedBox } from '@react-three/drei';
import * as THREE from 'three';
import { motion, AnimatePresence } from 'framer-motion';
import { Activity, Shield, Droplet, AlertTriangle, CheckCircle, XCircle, Zap, Move, Info } from 'lucide-react';

// --- Constants & Config ---
const THEME = {
  primary: '#06b6d4', // Cyan-500
  success: '#10b981', // Emerald-500
  danger: '#ef4444',  // Red-500
  warning: '#f59e0b', // Amber-500
  bg: '#0f172a',      // Slate-900
  glass: 'rgba(15, 23, 42, 0.85)',
};

const STEPS = [
  { id: 1, title: 'PPE Protocol', desc: 'Don sterile gloves before entering the hood.', icon: Shield },
  { id: 2, title: 'Hood Prep', desc: 'Wipe down surfaces with 70% Isopropyl Alcohol.', icon: Zap },
  { id: 3, title: 'Drug Withdrawal', desc: 'Insert needle and withdraw correct dosage (10ml).', icon: Droplet },
  { id: 4, title: 'Injection', desc: 'Inject drug into IV bag slowly.', icon: Activity },
  { id: 5, title: 'Labeling', desc: 'Apply patient label to the final product.', icon: Info },
];

// --- 3D Components ---

// Particle System for Laminar Flow
const LaminarFlow = ({ count = 1000 }) => {
  const mesh = useRef();
  const particles = useMemo(() => {
    const temp = [];
    for (let i = 0; i < count; i++) {
      const t = Math.random() * 100;
      const factor = 20 + Math.random() * 100;
      const speed = 0.01 + Math.random() / 200;
      const xFactor = -50 + Math.random() * 100;
      const yFactor = -50 + Math.random() * 100;
      const zFactor = -50 + Math.random() * 100;
      temp.push({ t, factor, speed, xFactor, yFactor, zFactor, mx: 0, my: 0 });
    }
    return temp;
  }, [count]);

  useFrame((state, delta) => {
    if (!mesh.current) return;
    particles.forEach((particle, i) => {
      let { t, factor, speed, xFactor, yFactor, zFactor } = particle;
      t = particle.t += speed / 2;
      const a = Math.cos(t) + Math.sin(t * 1) / 10;
      const b = Math.sin(t) + Math.cos(t * 2) / 10;
      const s = Math.cos(t);
      
      // Move particles down (Y axis) to simulate laminar flow
      particle.my -= speed * 10; 
      if (particle.my < -20) particle.my = 20; // Reset height

      mesh.current.setMatrixAt(i, new THREE.Matrix4().makeTranslation(
        xFactor + Math.cos(t) * 2, // Slight X drift
        particle.my,               // Y movement
        zFactor + Math.sin(t) * 2  // Slight Z drift
      ));
    });
    mesh.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh ref={mesh} args={[null, null, count]}>
      <sphereGeometry args={[0.02, 8, 8]} />
      <meshBasicMaterial color={THEME.primary} transparent opacity={0.3} />
    </instancedMesh>
  );
};

// Interactive Syringe
const Syringe = ({ position, rotation, onInteract, fillLevel, isDragging }) => {
  const group = useRef();
  
  // Animation for plunger movement based on fillLevel (0 to 1)
  const plungerY = useMemo(() => -1.5 + (fillLevel * 2.5), [fillLevel]);
  
  return (
    <group ref={group} position={position} rotation={rotation}>
      {/* Barrel */}
      <mesh position={[0, 0, 0]} castShadow receiveShadow>
        <cylinderGeometry args={[0.3, 0.3, 3, 32]} />
        <meshPhysicalMaterial 
          color="#ffffff" 
          transmission={0.6} 
          opacity={0.4} 
          transparent 
          roughness={0.1} 
          thickness={1}
        />
      </mesh>
      
      {/* Liquid Inside */}
      <mesh position={[0, -0.25 + (fillLevel * 1.25), 0]} scale={[0.9, fillLevel, 0.9]}>
        <cylinderGeometry args={[0.25, 0.25, 2.5, 32]} />
        <meshStandardMaterial color={fillLevel > 0 ? "#fbbf24" : "#fbbf24"} transparent opacity={0.8} />
      </mesh>

      {/* Plunger Rod */}
      <mesh position={[0, plungerY, 0]}>
        <cylinderGeometry args={[0.1, 0.1, 3, 16]} />
        <meshStandardMaterial color="#cbd5e1" />
      </mesh>

      {/* Plunger Head */}
      <mesh position={[0, plungerY + 1.4, 0]}>
        <cylinderGeometry args={[0.28, 0.28, 0.2, 32]} />
        <meshStandardMaterial color="#94a3b8" />
      </mesh>

      {/* Needle */}
      <mesh position={[0, 1.6, 0]} rotation={[Math.PI, 0, 0]}>
        <cylinderGeometry args={[0.02, 0.02, 1, 8]} />
        <meshStandardMaterial color="#94a3b8" metalness={0.8} roughness={0.2} />
      </mesh>
      
      {/* Interaction Zone (Invisible but clickable) */}
      <mesh 
        position={[0, 0, 0]} 
        visible={false}
        onPointerDown={(e) => { e.stopPropagation(); onInteract('start'); }}
        onPointerUp={(e) => { e.stopPropagation(); onInteract('end'); }}
        onPointerMove={(e) => { e.stopPropagation(); onInteract('move', e); }}
      >
        <boxGeometry args={[1.5, 4, 1.5]} />
      </mesh>
    </group>
  );
};

// Vial Component
const Vial = ({ position }) => {
  return (
    <group position={position}>
      {/* Glass Body */}
      <mesh position={[0, 0.5, 0]} castShadow receiveShadow>
        <cylinderGeometry args={[0.6, 0.6, 1.5, 32]} />
        <meshPhysicalMaterial 
          color="#e2e8f0" 
          transmission={0.9} 
          opacity={0.3} 
          transparent 
          roughness={0} 
        />
      </mesh>
      {/* Rubber Stopper */}
      <mesh position={[0, 1.3, 0]}>
        <cylinderGeometry args={[0.62, 0.62, 0.3, 32]} />
        <meshStandardMaterial color="#64748b" roughness={0.8} />
      </mesh>
      {/* Cap */}
      <mesh position={[0, 1.45, 0]}>
        <cylinderGeometry args={[0.65, 0.65, 0.1, 32]} />
        <meshStandardMaterial color={THEME.primary} metalness={0.5} roughness={0.2} />
      </mesh>
      {/* Liquid */}
      <mesh position={[0, 0.2, 0]}>
        <cylinderGeometry args={[0.55, 0.55, 1.2, 32]} />
        <meshStandardMaterial color="#f59e0b" transparent opacity={0.6} />
      </mesh>
    </group>
  );
};

// IV Bag Component
const IVBag = ({ position, fillLevel }) => {
  return (
    <group position={position}>
      {/* Bag Body - Deforming slightly with fill */}
      <mesh position={[0, 0, 0]} castShadow receiveShadow>
        <boxGeometry args={[2, 3, 0.5]} />
        <meshPhysicalMaterial 
          color="#ffffff" 
          transmission={0.7} 
          opacity={0.4} 
          transparent 
          roughness={0.2} 
        />
      </mesh>
      {/* Fluid inside bag */}
      <mesh position={[0, -0.5 + (fillLevel * 1.2), 0]} scale={[0.9, fillLevel, 0.8]}>
        <boxGeometry args={[1.8, 2.8, 0.4]} />
        <meshStandardMaterial color="#3b82f6" transparent opacity={0.6} />
      </mesh>
      {/* Ports */}
      <mesh position={[0, 1.6, 0]}>
        <cylinderGeometry args={[0.2, 0.2, 0.5, 16]} />
        <meshStandardMaterial color="#cbd5e1" />
      </mesh>
    </group>
  );
};

// Contamination Particles (Red floating dust)
const ContaminationParticles = ({ active }) => {
  const ref = useRef();
  useFrame((state) => {
    if (ref.current && active) {
      ref.current.rotation.y += 0.005;
      ref.current.position.y = Math.sin(state.clock.elapsedTime) * 0.5;
    }
  });

  if (!active) return null;

  return (
    <group ref={ref}>
      {[...Array(20)].map((_, i) => (
        <mesh key={i} position={[
          (Math.random() - 0.5) * 4,
          (Math.random() - 0.5) * 4,
          (Math.random() - 0.5) * 2
        ]}>
          <sphereGeometry args={[0.05]} />
          <meshBasicMaterial color={THEME.danger} />
        </mesh>
      ))}
    </group>
  );
};

// --- Main Application ---

const App = () => {
  // Game State
  const [step, setStep] = useState(1);
  const [xp, setXp] = useState(0);
  const [level, setLevel] = useState('Novice');
  const [glovesOn, setGlovesOn] = useState(false);
  const [hoodClean, setHoodClean] = useState(false);
  const [syringeFill, setSyringeFill] = useState(0); // 0 to 1
  const [bagFill, setBagFill] = useState(0); // 0 to 1
  const [contamination, setContamination] = useState(0); // 0 to 100
  const [isDraggingSyringe, setIsDraggingSyringe] = useState(false);
  const [notifications, setNotifications] = useState([]);
  const [shakeIntensity, setShakeIntensity] = useState(0);

  // Refs for interaction logic
  const lastPointerY = useRef(0);

  // --- Helpers ---
  const addNotification = (msg, type = 'info') => {
    const id = Date.now();
    setNotifications(prev => [...prev, { id, msg, type }]);
    setTimeout(() => {
      setNotifications(prev => prev.filter(n => n.id !== id));
    }, 3000);
  };

  const triggerContamination = (amount, reason) => {
    setContamination(prev => Math.min(prev + amount, 100));
    setShakeIntensity(1);
    setTimeout(() => setShakeIntensity(0), 500);
    addNotification(reason, 'error');
    // Haptic feedback if available
    if (navigator.vibrate) navigator.vibrate(200);
  };

  const checkStepCompletion = () => {
    // Step 1: PPE
    if (step === 1 && glovesOn) {
      setXp(prev => prev + 100);
      addNotification('PPE Donned Correctly', 'success');
      setStep(2);
    }
    // Step 2: Clean
    if (step === 2 && hoodClean) {
      setXp(prev => prev + 150);
      addNotification('Hood Sterilized', 'success');
      setStep(3);
    }
    // Step 3: Withdraw (Target 0.8 to 1.0 fill)
    if (step === 3 && syringeFill > 0.8) {
      setXp(prev => prev + 200);
      addNotification('Drug Withdrawn', 'success');
      setStep(4);
    }
    // Step 4: Inject
    if (step === 4 && bagFill > 0.9) {
      setXp(prev => prev + 250);
      addNotification('Injection Complete', 'success');
      setStep(5);
    }
    // Step 5: Label (Simulated by button)
    if (step === 5) {
        setXp(prev => prev + 300);
        addNotification('Simulation Complete!', 'success');
        if (navigator.vibrate) navigator.vibrate([100, 50, 100]);
    }
  };

  // --- Handlers ---

  const handleSyringeInteract = (action, e) => {
    if (step !== 3 && step !== 4) return;
    if (!glovesOn) {
      triggerContamination(20, 'Touching equipment without gloves!');
      return;
    }

    if (action === 'start') {
      setIsDraggingSyringe(true);
      lastPointerY.current = e.pointerY;
    } else if (action === 'end') {
      setIsDraggingSyringe(false);
    } else if (action === 'move' && isDraggingSyringe) {
      const delta = (lastPointerY.current - e.pointerY) * 0.01;
      lastPointerY.current = e.pointerY;
      
      if (step === 3) {
        // Withdrawing: Pulling up increases fill
        setSyringeFill(prev => Math.min(Math.max(prev + delta, 0), 1));
      } else if (step === 4) {
        // Injecting: Pushing down decreases syringe fill, increases bag fill
        const newSyringe = Math.max(syringeFill - delta, 0);
        const injectedAmount = syringeFill - newSyringe;
        setSyringeFill(newSyringe);
        setBagFill(prev => Math.min(prev + injectedAmount, 1));
      }
    }
  };

  const handleCleanHood = () => {
    if (!glovesOn) {
      triggerContamination(10, 'Gloves required for cleaning!');
      return;
    }
    // Simulate cleaning animation duration
    let progress = 0;
    const interval = setInterval(() => {
      progress += 5;
      if (progress >= 100) {
        clearInterval(interval);
        setHoodClean(true);
        checkStepCompletion();
      }
    }, 50);
  };

  const handleShakeDevice = () => {
    // Simulate device shake for testing contamination
    triggerContamination(30, 'Sudden movement detected! Aseptic technique broken.');
  };

  // Device Motion Listener for real shake
  useEffect(() => {
    const handleMotion = (event) => {
      const acc = event.accelerationIncludingGravity;
      if (!acc) return;
      const totalAcc = Math.sqrt(acc.x**2 + acc.y**2 + acc.z**2);
      if (totalAcc > 15) { // Threshold for shake
        triggerContamination(10, 'Excessive movement detected.');
      }
    };
    window.addEventListener('devicemotion', handleMotion);
    return () => window.removeEventListener('devicemotion', handleMotion);
  }, []);

  // Level Up Logic
  useEffect(() => {
    if (xp > 500) setLevel('Advanced Practitioner');
    if (xp > 1000) setLevel('Oncology Expert');
  }, [xp]);

  // --- Render ---

  const currentStepData = STEPS.find(s => s.id === step);
  const Icon = currentStepData?.icon;

  return (
    <div className="relative w-full h-screen overflow-hidden bg-slate-900 text-white font-sans select-none touch-none">
      
      {/* Background Image Layer */}
      <div 
        className="absolute inset-0 z-0 opacity-40 bg-cover bg-center transition-transform duration-100"
        style={{ 
          backgroundImage: `url('https://image.qwenlm.ai/public_source/4ce5aacd-1274-4f9a-b30a-402021029ba2/14d8ee9a1-037f-455d-bbc3-c0d0b7d58945.png')`,
          transform: `translate(${Math.random() * shakeIntensity * 10 - 5}px, ${Math.random() * shakeIntensity * 10 - 5}px)`
        }}
      />
      
      {/* Contamination Overlay */}
      {contamination > 0 && (
        <div className="absolute inset-0 z-10 pointer-events-none bg-red-500/10 mix-blend-overlay" />
      )}

      {/* 3D Scene Layer */}
      <div className="absolute inset-0 z-0">
        <Canvas shadows camera={{ position: [0, 2, 6], fov: 50 }}>
          <color attach="background" args={['#0f172a']} />
          <fog attach="fog" args={['#0f172a', 5, 20]} />
          
          <ambientLight intensity={0.5} />
          <spotLight position={[5, 10, 5]} angle={0.5} penumbra={1} intensity={2} castShadow />
          <pointLight position={[-5, 5, -5]} intensity={1} color={THEME.primary} />
          
          <Environment preset="city" />

          <Suspense fallback={null}>
            <group position={[0, -1, 0]}>
              {/* Laminar Flow Particles */}
              <LaminarFlow />
              
              {/* Contamination Particles */}
              <ContaminationParticles active={contamination > 20} />

              {/* Equipment */}
              <Vial position={[-2, 0, 0]} />
              <IVBag position={[2, 0, 0]} fillLevel={bagFill} />
              
              {/* Interactive Syringe */}
              <Syringe 
                position={[0, 0, 1]} 
                rotation={[0, 0, 0]} 
                fillLevel={syringeFill}
                isDragging={isDraggingSyringe}
                onInteract={handleSyringeInteract}
              />

              <ContactShadows resolution={1024} scale={10} blur={2} opacity={0.5} far={10} color="#000000" />
            </group>
          </Suspense>
          
          <OrbitControls 
            enableZoom={false} 
            enablePan={false} 
            minPolarAngle={Math.PI / 3} 
            maxPolarAngle={Math.PI / 2} 
          />
        </Canvas>
      </div>

      {/* UI HUD Layer */}
      <div className="absolute inset-0 z-20 pointer-events-none flex flex-col justify-between p-4 md:p-8">
        
        {/* Top Bar: Stats */}
        <div className="flex justify-between items-start pointer-events-auto">
          <div className="flex flex-col gap-2">
            <div className="bg-slate-800/80 backdrop-blur-md border border-slate-700 p-3 rounded-xl flex items-center gap-3 shadow-lg">
              <div className="bg-cyan-500/20 p-2 rounded-lg text-cyan-400">
                <Activity size={24} />
              </div>
              <div>
                <div className="text-xs text-slate-400 uppercase tracking-wider">Simulation Level</div>
                <div className="font-bold text-lg">{level}</div>
              </div>
            </div>
            
            <div className="bg-slate-800/80 backdrop-blur-md border border-slate-700 p-3 rounded-xl flex items-center gap-3 shadow-lg">
              <div className="bg-amber-500/20 p-2 rounded-lg text-amber-400">
                <Zap size={24} />
              </div>
              <div>
                <div className="text-xs text-slate-400 uppercase tracking-wider">XP Points</div>
                <div className="font-bold text-lg">{xp}</div>
              </div>
            </div>
          </div>

          {/* Contamination Meter */}
          <div className="w-32 bg-slate-800/80 backdrop-blur-md border border-slate-700 p-3 rounded-xl shadow-lg">
            <div className="flex justify-between text-xs mb-1">
              <span className="text-slate-400">STERILITY</span>
              <span className={contamination > 20 ? 'text-red-400' : 'text-emerald-400'}>
                {100 - contamination}%
              </span>
            </div>
            <div className="h-2 bg-slate-700 rounded-full overflow-hidden">
              <motion.div 
                className={`h-full ${contamination > 50 ? 'bg-red-500' : 'bg-emerald-500'}`}
                initial={{ width: '100%' }}
                animate={{ width: `${100 - contamination}%` }}
              />
            </div>
          </div>
        </div>

        {/* Center: Step Indicator & Warnings */}
        <div className="absolute top-1/4 left-0 right-0 flex justify-center pointer-events-none">
           <AnimatePresence mode="wait">
            <motion.div
              key={step}
              initial={{ opacity: 0, y: -20 }}
              animate={{ opacity: 1, y: 0 }}
              exit={{ opacity: 0, y: 20 }}
              className="bg-slate-900/90 backdrop-blur-xl border border-slate-700 px-6 py-4 rounded-2xl shadow-2xl text-center max-w-md mx-4"
            >
              <div className="flex items-center justify-center gap-3 mb-2">
                <Icon className="text-cyan-400" size={28} />
                <h2 className="text-xl font-bold text-white">{currentStepData?.title}</h2>
              </div>
              <p className="text-slate-300">{currentStepData?.desc}</p>
              
              {/* Progress Bar for current step actions */}
              {(step === 3 || step === 4) && (
                <div className="mt-4 h-1 w-full bg-slate-700 rounded-full overflow-hidden">
                  <motion.div 
                    className="h-full bg-cyan-500"
                    animate={{ width: step === 3 ? `${syringeFill * 100}%` : `${bagFill * 100}%` }}
                  />
                </div>
              )}
            </motion.div>
          </AnimatePresence>
        </div>

        {/* Bottom: Controls */}
        <div className="flex flex-col gap-4 pointer-events-auto">
          
          {/* Notifications Toast */}
          <div className="absolute bottom-32 left-0 right-0 flex flex-col items-center gap-2 pointer-events-none">
            <AnimatePresence>
              {notifications.map(n => (
                <motion.div
                  key={n.id}
                  initial={{ opacity: 0, scale: 0.9, y: 20 }}
                  animate={{ opacity: 1, scale: 1, y: 0 }}
                  exit={{ opacity: 0, scale: 0.9, y: -20 }}
                  className={`px-4 py-2 rounded-lg shadow-lg flex items-center gap-2 text-sm font-bold ${
                    n.type === 'error' ? 'bg-red-500 text-white' : 
                    n.type === 'success' ? 'bg-emerald-500 text-white' : 
                    'bg-slate-700 text-white'
                  }`}
                >
                  {n.type === 'error' && <XCircle size={16} />}
                  {n.type === 'success' && <CheckCircle size={16} />}
                  {n.msg}
                </motion.div>
              ))}
            </AnimatePresence>
          </div>

          {/* Action Buttons */}
          <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
            
            {/* Step 1: PPE */}
            {step === 1 && (
              <motion.button
                whileTap={{ scale: 0.95 }}
                onClick={() => { setGlovesOn(true); checkStepCompletion(); }}
                className="col-span-2 bg-cyan-600 hover:bg-cyan-500 text-white p-4 rounded-xl font-bold text-lg shadow-lg flex items-center justify-center gap-2"
              >
                <Shield size={24} /> DON GLOVES
              </motion.button>
            )}

            {/* Step 2: Clean */}
            {step === 2 && (
              <motion.button
                whileTap={{ scale: 0.95 }}
                onClick={handleCleanHood}
                disabled={hoodClean}
                className={`col-span-2 p-4 rounded-xl font-bold text-lg shadow-lg flex items-center justify-center gap-2 ${
                  hoodClean ? 'bg-emerald-600 text-white' : 'bg-amber-600 hover:bg-amber-500 text-white'
                }`}
              >
                <Zap size={24} /> {hoodClean ? 'HOOD STERILE' : 'WIPE SURFACES'}
              </motion.button>
            )}

            {/* Step 3 & 4: Instructions */}
            {(step === 3 || step === 4) && (
              <div className="col-span-2 bg-slate-800/80 border border-slate-600 p-4 rounded-xl flex items-center justify-center text-slate-300">
                <Move size={20} className="mr-2 animate-pulse" />
                {step === 3 ? 'DRAG PLUNGER UP TO WITHDRAW' : 'DRAG PLUNGER DOWN TO INJECT'}
              </div>
            )}

            {/* Step 5: Label */}
            {step === 5 && (
              <motion.button
                whileTap={{ scale: 0.95 }}
                onClick={checkStepCompletion}
                className="col-span-2 bg-cyan-600 hover:bg-cyan-500 text-white p-4 rounded-xl font-bold text-lg shadow-lg flex items-center justify-center gap-2"
              >
                <Info size={24} /> APPLY LABEL & FINISH
              </motion.button>
            )}

            {/* Debug / Extra Controls */}
            <button 
              onClick={handleShakeDevice}
              className="col-span-2 md:col-span-2 bg-slate-800 hover:bg-slate-700 text-slate-400 p-3 rounded-xl text-sm font-medium border border-slate-700"
            >
              SIMULATE SHAKE (TEST)
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

const root = createRoot(document.getElementById('root'));
root.render(<App />);
