import React, { useState, useEffect, useRef, useCallback } from "react";
import { View, ScrollView, StyleSheet, Text, TouchableOpacity, TextInput, Alert, ActivityIndicator, Platform, Modal } from "react-native";
import AsyncStorage from "@react-native-async-storage/async-storage";
import { Feather, MaterialCommunityIcons, Ionicons } from "@expo/vector-icons";
import ConfettiCannon from "react-native-confetti-cannon";
import NetInfo from "@react-native-community/netinfo";
import firebase from "firebase/compat/app";
import "firebase/compat/firestore";
import { SafeAreaView, SafeAreaProvider } from "react-native-safe-area-context";
import RNPickerSelect from "react-native-picker-select";

// ========== CONSTANTS & HELPERS ==========

const APP_VERSION = "1.0.1";
const BUILD_DATE = "2025-06-07";
const CURRENT_DATE = "2025-06-07";
const CURRENT_TIME = "14:45:25";

const commonUKBreeds = [
  "Labrador Retriever",
  "Cocker Spaniel",
  "French Bulldog",
  "Bulldog",
  "Dachshund",
  "German Shepherd",
  "Jack Russell Terrier",
  "Staffordshire Bull Terrier",
  "Border Collie",
  "Golden Retriever"
];

const breedTips = {
  "Labrador Retriever": [
    "Labradors love water – include swimming in their routine.",
    "They respond well to food rewards but watch their weight."
  ],
  "Cocker Spaniel": [
    "Spaniels are energetic and need regular, varied exercise.",
    "Mental stimulation is crucial for this clever breed."
  ],
  "French Bulldog": [
    "Short walks suit their breathing; avoid heat.",
    "They thrive on companionship and gentle training."
  ],
  "Bulldog": [
    "Keep sessions short and allow rest breaks.",
    "Monitor for overheating, especially in warm weather."
  ],
  "Dachshund": [
    "Protect their back: avoid stairs and jumping.",
    "Short, positive training sessions work best."
  ],
  "German Shepherd": [
    "GSDs need jobs: try agility or advanced obedience.",
    "Early and ongoing socialisation is key."
  ],
  "Jack Russell Terrier": [
    "Channel their energy into games and puzzles.",
    "Consistent boundaries are essential."
  ],
  "Staffordshire Bull Terrier": [
    "They excel with positive reinforcement.",
    "Plenty of social play helps prevent boredom."
  ],
  "Border Collie": [
    "They thrive on advanced tricks and mental work.",
    "Provide daily tasks to keep them busy."
  ],
  "Golden Retriever": [
    "Retrievers love to carry and fetch – use this in games.",
    "Social, gentle training is most effective."
  ]
};

const firebaseConfig = {
  apiKey: "AIzaSyDPT2FZbLupAd9F_V1CB87i5CUl2oaULLg",
  authDomain: "ludos-training-tracker.firebaseapp.com",
  projectId: "ludos-training-tracker",
  storageBucket: "ludos-training-tracker.appspot.com",
  messagingSenderId: "599255592851",
  appId: "1:599255592851:android:5c758bc134326f9b40c266"
};

if (!firebase.apps.length) {
  firebase.initializeApp(firebaseConfig);
  firebase.firestore().enablePersistence({ synchronizeTabs: false }).catch(function(err) { 
    console.error("Firestore persistence error:", err); 
  });
}
const db = firebase.firestore();

const trainingStages = {
  1: { name: "Foundation (Weeks 1-4)", range: "8-12 weeks", color: "#bbf7d0" },
  2: { name: "Impulse Control (Weeks 5-16)", range: "12-16 weeks", color: "#dbeafe" },
  3: { name: "Building Obedience (Weeks 17-26)", range: "4-6 months", color: "#ede9fe" },
  4: { name: "Adolescent Phase (Weeks 27+)", range: "6-12 months", color: "#fed7aa" }
};

const weeklyPlans = {
  1: ["Name recognition", "Crate training", "Potty training"],
  2: ["Sit & Down", "Handling paws/ears/mouth", "Intro to leash"],
  3: ["Meet calm dogs", "New surfaces & sounds", "Short training sessions"],
  4: ["Basic recall (indoors)", "Continue potty/crate", "Social walk (carry if needed)"],
  5: ["Wait & Leave it", "Loose-leash walking", "Puzzle toy intro"],
  6: ["Sit-Stay & Down-Stay", "Recall indoors", "Traffic & bike exposure"],
  7: ["Tug with drop-it", "Find-it game", "New sounds/social areas"],
  8: ["Reinforce all basics", "Puzzle time", "Crate review"],
  9: ["Recall w/ distractions", "Advanced stays", "Marker word training"],
  10: ["Place command", "Puzzle toy advanced", "Spin trick"],
  11: ["Scent games", "Shake trick", "Review leash & crate"],
  12: ["Bow trick", "New distractions", "Clicker practice"],
  13: ["Mix tricks: combo day", "Trail walk intro", "Social calm practice"],
  14: ["Stays with duration", "Heel in quiet area", "Slow feeder puzzle"],
  15: ["Clicker games", "Play with purpose", "Place & Stay combo"],
  16: ["Dog-friendly cafe visit", "New location recall", "Tug with control"]
};

const dailyRoutine = {
  morning: ["Potty break", "Breakfast", "Basic commands review"],
  midday: ["Primary training focus", "Socialization/Exposure", "Mental stimulation"],
  evening: ["Recall practice", "Command reinforcement", "Cool down"],
  play: ["Structured play", "Bonding time", "Free play"]
};

const MAX_WEEKS = 16;

// Utility functions
const getCurrentStage = week => {
  if (week <= 4) return 1;
  if (week <= 16) return 2;
  if (week <= 26) return 3;
  return 4;
};

const isValidDate = (dateString) => {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(dateString)) return false;
  const date = new Date(dateString + 'T00:00:00.000Z');
  return !isNaN(date) && dateString === date.toISOString().slice(0, 10);
};

const deepEqual = (obj1, obj2) => {
  if (obj1 === obj2) return true;
  if (obj1 == null || obj2 == null) return false;
  if (typeof obj1 !== typeof obj2) return false;
  
  if (typeof obj1 !== 'object') return obj1 === obj2;
  
  const keys1 = Object.keys(obj1);
  const keys2 = Object.keys(obj2);
  
  if (keys1.length !== keys2.length) return false;
  
  for (let key of keys1) {
    if (!keys2.includes(key)) return false;
    if (!deepEqual(obj1[key], obj2[key])) return false;
  }
  
  return true;
};

function calcDogAge(dob, weekOffset = 0, nowDate) {
  if (!dob) return "";
  const dobDate = new Date(dob);
  if (isNaN(dobDate)) return "";
  let baseTime = nowDate ? nowDate.getTime() : Date.now();
  let virtualDate = new Date(dobDate.getTime() + weekOffset * 7 * 24 * 60 * 60 * 1000);
  let diff = baseTime - virtualDate.getTime();
  if (diff < 0) diff = 0;
  let ageDate = new Date(diff);
  let years = ageDate.getUTCFullYear() - 1970;
  let months = ageDate.getUTCMonth();
  let days = ageDate.getUTCDate() - 1;
  let weeks = Math.floor(days / 7);
  let ageStr = "";
  if (years > 0) ageStr += `${years}y `;
  if (months > 0) ageStr += `${months}m `;
  if (weeks > 0) ageStr += `${weeks}w`;
  if (!ageStr) ageStr = "0w";
  return ageStr.trim();
}

async function updateDogInfoStorageAndFirestore(familyId, name, breed, dob) {
  await AsyncStorage.setItem("dogName", name);
  await AsyncStorage.setItem("dogBreed", breed);
  await AsyncStorage.setItem("dogDob", dob);
  if (familyId) {
    await db.collection("families").doc(familyId).set(
      { dogName: name, dogBreed: breed, dogDob: dob },
      { merge: true }
    );
  }
}

// ========== ERROR BOUNDARY ==========
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('App Error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <View style={styles.errorContainer}>
          <Feather name="alert-triangle" size={48} color="#ef4444" />
          <Text style={styles.errorTitle}>Something went wrong</Text>
          <Text style={styles.errorText}>Please restart the app</Text>
          <TouchableOpacity 
            style={styles.errorButton}
            onPress={() => this.setState({ hasError: false })}
          >
            <Text style={styles.errorButtonText}>Try Again</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return this.props.children;
  }
}

// ========== MAIN APP COMPONENT ==========

function MainApp() {
  const [isOffline, setIsOffline] = useState(false);
  
  // Core state
  const [familyId, setFamilyId] = useState(null);
  const [dogName, setDogName] = useState(null);
  const [dogBreed, setDogBreed] = useState(null);
  const [dogDob, setDogDob] = useState(null);
  
  // Input states
  const [inputId, setInputId] = useState("");
  const [inputDog, setInputDog] = useState("");
  const [inputBreed, setInputBreed] = useState("");
  const [inputBreedOther, setInputBreedOther] = useState("");
  const [inputDob, setInputDob] = useState("");
  
  // UI states
  const [showDogInput, setShowDogInput] = useState(false);
  const [familyDogLoading, setFamilyDogLoading] = useState(true);
  const [checkingFamilyId, setCheckingFamilyId] = useState(false);
  const [loading, setLoading] = useState(true);
  const [syncing, setSyncing] = useState(false);
  
  // Edit modal states
  const [editDogModal, setEditDogModal] = useState(false);
  const [editDogName, setEditDogName] = useState("");
  const [editDogBreed, setEditDogBreed] = useState("");
  const [editDogDob, setEditDogDob] = useState("");
  const [editDogBreedOther, setEditDogBreedOther] = useState("");
  
  // App data states
  const [currentWeek, setCurrentWeek] = useState(1);
  const [completedActivities, setCompletedActivities] = useState([]);
  const [dailyNotes, setDailyNotes] = useState({});
  const [viewMode, setViewMode] = useState("daily");
  const [selectedDate, setSelectedDate] = useState(() => CURRENT_DATE);
  const [completedDailyByDate, setCompletedDailyByDate] = useState({});
  
  // Family and notes states
  const [family, setFamily] = useState([
    { id: 1, name: "Primary Trainer", editing: false },
    { id: 2, name: "Family Member 2", editing: false },
    { id: 3, name: "Family Member 3", editing: false }
  ]);
  const [newFamilyName, setNewFamilyName] = useState("");
  const [editFamilyName, setEditFamilyName] = useState({});
  const [editingMemberId, setEditingMemberId] = useState(null);
  
  const [sharedNotes, setSharedNotes] = useState([]);
  const [newSharedNote, setNewSharedNote] = useState("");
  const [notesLoading, setNotesLoading] = useState(false);
  const [editingNoteId, setEditingNoteId] = useState(null);
  const [editingNoteText, setEditingNoteText] = useState("");
  const [currentUserName, setCurrentUserName] = useState("Wiewioreq");
  
  // Celebration states
  const [showDailyConfetti, setShowDailyConfetti] = useState(false);
  const [showWeeklyConfetti, setShowWeeklyConfetti] = useState(false);
  const [currentDogAge, setCurrentDogAge] = useState(() => calcDogAge(dogDob, 0, new Date()));
  
  // Refs for cleanup and optimization
  const syncTimeout = useRef();
  const lastSyncedData = useRef({});
  const dobInputRef = useRef(null);

  // Network status effect
  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOffline(!state.isConnected);
    });
    NetInfo.fetch().then(state => setIsOffline(!state.isConnected));
    return () => unsubscribe();
  }, []);

  // Cleanup timeouts on unmount
  useEffect(() => {
    return () => {
      if (syncTimeout.current) {
        clearTimeout(syncTimeout.current);
      }
    };
  }, []);

  // Initial data loading
  useEffect(() => {
    (async () => {
      try {
        const id = await AsyncStorage.getItem("familyId");
        const dog = await AsyncStorage.getItem("dogName");
        const breed = await AsyncStorage.getItem("dogBreed");
        const dob = await AsyncStorage.getItem("dogDob");
        const username = await AsyncStorage.getItem("memberName");
        
        if (id) setFamilyId(id);
        if (dog) setDogName(dog);
        if (breed) setDogBreed(breed);
        if (dob) setDogDob(dob);
        if (username) setCurrentUserName(username);
      } catch (e) {
        console.error("Error loading dog/family data:", e);
      }
      
      try {
        const cdbd = await AsyncStorage.getItem("completedDailyByDate");
        if (cdbd) {
          const parsedData = JSON.parse(cdbd);
          console.log("Loaded daily task data from storage:", parsedData);
          setCompletedDailyByDate(parsedData);
        }
      } catch (e) {
        console.error("Failed to parse completedDailyByDate:", e);
      } finally {
        setFamilyDogLoading(false);
      }
    })();
  }, []);

  // Set current user name in AsyncStorage when it changes
  useEffect(() => {
    if (currentUserName) {
      AsyncStorage.setItem("memberName", currentUserName);
    }
  }, [currentUserName]);

  // Optimized Firebase sync with deep comparison
  const syncToFirebase = useCallback(async (data) => {
    if (!familyId || !data) return;
    
    // Only sync if data actually changed
    if (deepEqual(lastSyncedData.current, data)) {
      setSyncing(false);
      return;
    }
    
    try {
      setSyncing(true);
      await db.collection("families").doc(familyId).set(data, { merge: true });
      lastSyncedData.current = { ...data };
      console.log("Data synced to Firebase successfully");
    } catch (e) {
      console.error("Failed to sync to Firebase:", e);
      if (!isOffline) {
        Alert.alert("Sync Error", "Failed to save changes. Please check your connection.");
      }
    } finally {
      setSyncing(false);
    }
  }, [familyId, isOffline]);

  // Consolidated debounced sync effect
  useEffect(() => {
    if (loading || !familyId) return;
    
    const dataToSync = {
      currentWeek,
      completedActivities,
      dailyNotes,
      dogName,
      dogBreed,
      dogDob,
      family,
      sharedNotes,
      completedDailyByDate
    };
    
    // Clear existing timeout
    if (syncTimeout.current) {
      clearTimeout(syncTimeout.current);
    }
    
    // Debounce sync by 500ms
    syncTimeout.current = setTimeout(() => {
      syncToFirebase(dataToSync);
      // Also save to AsyncStorage for offline access
      AsyncStorage.setItem("completedDailyByDate", JSON.stringify(completedDailyByDate))
        .catch(e => console.error("AsyncStorage save error", e));
    }, 500);
    
  }, [currentWeek, completedActivities, dailyNotes, dogName, dogBreed, dogDob, family, sharedNotes, completedDailyByDate, syncToFirebase, loading, familyId]);

  // Firebase snapshot listener with optimized updates
  useEffect(() => {
    if (!familyId) return;
    
    setLoading(true);
    setNotesLoading(true);
    
    const unsubscribe = db
      .collection("families")
      .doc(familyId)
      .onSnapshot(
        docSnap => {
          if (docSnap.exists) {
            const data = docSnap.data();
            
            // Only update state if values actually changed (reduces re-renders)
            if (data.currentWeek !== undefined && data.currentWeek !== currentWeek) {
              setCurrentWeek(data.currentWeek);
            }
            
            if (data.completedActivities && !deepEqual(data.completedActivities, completedActivities)) {
              setCompletedActivities(data.completedActivities);
            }
            
            if (data.dailyNotes && !deepEqual(data.dailyNotes, dailyNotes)) {
              setDailyNotes(data.dailyNotes);
            }
            
            if (data.family && !deepEqual(data.family, family)) {
              setFamily(data.family);
            }
            
            if (data.sharedNotes && !deepEqual(data.sharedNotes, sharedNotes)) {
              setSharedNotes(data.sharedNotes);
            }
            
            if (data.dogName && data.dogName !== dogName) {
              setDogName(data.dogName);
              AsyncStorage.setItem("dogName", data.dogName);
            }
            
            if (data.dogBreed && data.dogBreed !== dogBreed) {
              setDogBreed(data.dogBreed);
              AsyncStorage.setItem("dogBreed", data.dogBreed);
            }
            
            if (data.dogDob && data.dogDob !== dogDob) {
              setDogDob(data.dogDob);
              AsyncStorage.setItem("dogDob", data.dogDob);
            }
            
            if (data.completedDailyByDate && !deepEqual(data.completedDailyByDate, completedDailyByDate)) {
              setCompletedDailyByDate(data.completedDailyByDate);
              AsyncStorage.setItem("completedDailyByDate", JSON.stringify(data.completedDailyByDate))
                .catch(e => console.error("AsyncStorage save error", e));
            }
            
            // Update lastSyncedData to prevent immediate re-sync
            lastSyncedData.current = {
              currentWeek: data.currentWeek ?? 1,
              completedActivities: data.completedActivities ?? [],
              dailyNotes: data.dailyNotes ?? {},
              dogName: data.dogName,
              dogBreed: data.dogBreed,
              dogDob: data.dogDob,
              family: data.family ?? [],
              sharedNotes: data.sharedNotes ?? [],
              completedDailyByDate: data.completedDailyByDate ?? {}
            };
          }
          setLoading(false);
          setNotesLoading(false);
        },
        err => {
          setLoading(false);
          setNotesLoading(false);
          console.error("Firestore snapshot error:", err);
          if (!isOffline) {
            Alert.alert("Sync error", "Could not sync with Firebase: " + err.message);
          }
        }
      );
    
    return unsubscribe;
  }, [familyId]);

  // Dog age calculation effect
  useEffect(() => {
    if (!dogDob) return;
    const interval = setInterval(() => {
      setCurrentDogAge(calcDogAge(dogDob, 0, new Date()));
    }, 60 * 1000);
    setCurrentDogAge(calcDogAge(dogDob, 0, new Date()));
    return () => clearInterval(interval);
  }, [dogDob]);

  // Family ID submission
  const handleFamilyIdSubmit = async () => {
    if (!inputId.trim()) return;
    setCheckingFamilyId(true);
    const famId = inputId.trim();
    
    try {
      const docSnap = await db.collection("families").doc(famId).get();
      if (docSnap.exists) {
        const data = docSnap.data();
        if (data.dogName && data.dogBreed && data.dogDob) {
          await AsyncStorage.setItem("familyId", famId);
          await AsyncStorage.setItem("dogName", data.dogName);
          await AsyncStorage.setItem("dogBreed", data.dogBreed);
          await AsyncStorage.setItem("dogDob", data.dogDob);
          setFamilyId(famId);
          setDogName(data.dogName);
          setDogBreed(data.dogBreed);
          setDogDob(data.dogDob);
          setCheckingFamilyId(false);
          setShowDogInput(false);
        } else {
          Alert.alert("Family ID Exists", "This Family ID exists but is missing some dog info. Please contact your admin.");
          setCheckingFamilyId(false);
        }
      } else {
        Alert.alert(
          "Create New Family?",
          `No family found for ID "${famId}". Do you want to create a new family with this ID?`,
          [
            { text: "Cancel", style: "cancel", onPress: () => setCheckingFamilyId(false) },
            { text: "Create", style: "default", onPress: () => { setShowDogInput(true); setCheckingFamilyId(false); } }
          ]
        );
      }
    } catch (e) {
      Alert.alert("Network error", e.message);
      setCheckingFamilyId(false);
    }
  };

  // Dog details submission with improved validation
  const handleDogDetailsSubmit = async () => {
    const breedToSave = inputBreed === "Other" ? inputBreedOther.trim() : inputBreed;
    
    if (!inputId.trim() || !inputDog.trim() || !breedToSave || !inputDob.trim()) {
      Alert.alert("Missing Information", "Please fill in all required fields.");
      return;
    }
    
    if (inputBreed === "Other" && !inputBreedOther.trim()) {
      Alert.alert("Breed Required", "Please specify the breed.");
      return;
    }
    
    if (!isValidDate(inputDob.trim())) {
      Alert.alert("Invalid Date", "Please enter a valid date in YYYY-MM-DD format.");
      return;
    }
    
    const famId = inputId.trim();
    const dName = inputDog.trim();
    const dBreed = breedToSave;
    const dDob = inputDob.trim();
    
    try {
      await db.collection("families").doc(famId).set({
        dogName: dName,
        dogBreed: dBreed,
        dogDob: dDob,
        family: [
          { id: 1, name: "Primary Trainer", editing: false },
          { id: 2, name: "Family Member 2", editing: false },
          { id: 3, name: "Family Member 3", editing: false }
        ]
      }, { merge: true });
      
      await AsyncStorage.setItem("familyId", famId);
      await AsyncStorage.setItem("dogName", dName);
      await AsyncStorage.setItem("dogBreed", dBreed);
      await AsyncStorage.setItem("dogDob", dDob);
      
      setFamilyId(famId);
      setDogName(dName);
      setDogBreed(dBreed);
      setDogDob(dDob);
      setShowDogInput(false);
    } catch (e) {
      Alert.alert("Network error", e.message);
    }
  };

  // Daily activity toggle with optimistic UI updates
  const today = CURRENT_DATE;
  const completedToday = completedDailyByDate[today] || [];
  
  const toggleDailyActivity = useCallback((timeSlot, activity) => {
    const key = `daily-${timeSlot}-${activity}`;
    
    setCompletedDailyByDate(prev => {
      const prevForToday = prev[today] || [];
      const isCompleted = prevForToday.includes(key);
      
      const updatedData = {
        ...prev,
        [today]: isCompleted 
          ? prevForToday.filter(k => k !== key)
          : [...prevForToday, key]
      };
      
      return updatedData;
    });
  }, [today]);

  // Notes handling with direct Firebase save
  const addNote = useCallback(async (date, note) => {
    const updatedNotes = { ...dailyNotes, [date]: note };
    setDailyNotes(updatedNotes);
    
    // Direct save to Firebase for immediate persistence
    if (familyId) {
      try {
        await db.collection("families").doc(familyId).set(
          { dailyNotes: updatedNotes }, 
          { merge: true }
        );
      } catch (e) {
        console.error("Failed to save daily note:", e);
      }
    }
  }, [dailyNotes, familyId]);

  const handleAddSharedNote = async () => {
    const trimmed = newSharedNote.trim();
    if (!trimmed) return;
    
    const author = currentUserName || (family && family.length > 0 ? family[0].name : "You");
    
    const noteObj = {
      id: Date.now().toString() + Math.random().toString(36).slice(2),
      text: trimmed,
      author,
      timestamp: new Date().toISOString(),
      authorId: currentUserName
    };
    
    const updatedNotes = [noteObj, ...(sharedNotes || [])];
    
    // Optimistic update
    setSharedNotes(updatedNotes);
    setNewSharedNote("");
    
    // Save to Firebase
    try {
      await db.collection("families").doc(familyId).set(
        { sharedNotes: updatedNotes },
        { merge: true }
      );
    } catch (e) {
      console.error("Failed to save shared note:", e);
      Alert.alert("Error", "Failed to save note. Please try again.");
      // Revert optimistic update on error
      setSharedNotes(sharedNotes);
      setNewSharedNote(trimmed);
    }
  };

  const saveNoteEdit = async (noteId) => {
    if (!editingNoteText.trim()) {
      Alert.alert("Cannot save", "Note cannot be empty.");
      return;
    }

    const updatedNotes = sharedNotes.map(note =>
      note.id === noteId ? { 
        ...note, 
        text: editingNoteText.trim(),
        edited: true,
        editTimestamp: new Date().toISOString()
      } : note
    );
    
    // Optimistic update
    setSharedNotes(updatedNotes);
    setEditingNoteId(null);
    setEditingNoteText("");
    
    try {
      await db.collection("families").doc(familyId).set(
        { sharedNotes: updatedNotes },
        { merge: true }
      );
    } catch (e) {
      console.error("Failed to update note:", e);
      Alert.alert("Error", "Failed to update the note. Please try again.");
    }
  };

  // Activity calculations
  const allDailyKeys = Object.entries(dailyRoutine)
    .flatMap(([slot, acts]) =>
      acts.map(activity => `daily-${slot}-${activity}`)
    );
  const completedDaily = completedToday.length;
  const dailyRate = allDailyKeys.length > 0 ? completedDaily / allDailyKeys.length : 0;
  
  const completionRate = (() => {
    const currentWeekActivities = weeklyPlans[currentWeek] || [];
    const completed = currentWeekActivities.filter(act =>
      completedActivities.includes(`${currentWeek}-${act}`)
    ).length;
    return currentWeekActivities.length
      ? completed / currentWeekActivities.length
      : 0;
  })();

  // Confetti effects
  useEffect(() => {
    if (dailyRate === 1 && !showDailyConfetti) {
      setShowDailyConfetti(true);
      setTimeout(() => setShowDailyConfetti(false), 3500);
    }
  }, [dailyRate, showDailyConfetti]);

  useEffect(() => {
    const currentWeekActivities = weeklyPlans[currentWeek] || [];
    const allDone = currentWeekActivities.every(act =>
      completedActivities.includes(`${currentWeek}-${act}`)
    );
    if (allDone && !showWeeklyConfetti && currentWeekActivities.length > 0) {
      setShowWeeklyConfetti(true);
      setTimeout(() => setShowWeeklyConfetti(false), 3500);
    }
  }, [currentWeek, completedActivities, showWeeklyConfetti]);

  // Activity toggle
  const toggleActivity = useCallback((week, activity) => {
    let key;
    if (week === "milestone") {
      key = `milestone-${activity}`;
    } else {
      key = `${week}-${activity}`;
    }
    setCompletedActivities(prev =>
      prev.includes(key) ? prev.filter(k => k !== key) : [...prev, key]
    );
  }, []);

  const currentStage = getCurrentStage(currentWeek);

  // Family member management
  const handleEditFamilyMember = useCallback(id => {
    setEditingMemberId(id);
    setEditFamilyName({ ...editFamilyName, [id]: family.find(f => f.id === id)?.name || "" });
  }, [editFamilyName, family]);

  const handleSaveFamilyMember = useCallback((id, newName) => {
    setFamily(family =>
      family.map(f =>
        f.id === id ? { ...f, name: newName || "Unnamed", editing: false } : f
      )
    );
    setEditingMemberId(null);
  }, []);

  const handleRemoveFamilyMember = useCallback(id => {
    Alert.alert(
      "Remove member",
      "Are you sure you want to remove this family member?",
      [
        { text: "Cancel", style: "cancel" },
        {
          text: "Remove",
          style: "destructive",
          onPress: () => setFamily(family => family.filter(f => f.id !== id))
        }
      ]
    );
  }, []);

  const handleAddFamilyMember = useCallback(() => {
    if (!newFamilyName.trim()) return;
    setFamily(family => [...family, { id: Date.now(), name: newFamilyName, editing: false }]);
    setNewFamilyName("");
  }, [newFamilyName]);

  // Dog editing modal with improved validation
  const openEditDogModal = useCallback(() => {
    try {
      const currentName = dogName || "";
      
      let currentBreed = "Other";
      let currentBreedOther = "";
      
      if (dogBreed) {
        if (commonUKBreeds.includes(dogBreed)) {
          currentBreed = dogBreed;
        } else {
          currentBreed = "Other";
          currentBreedOther = dogBreed;
        }
      }
      
      const currentDob = dogDob || "";
      
      setEditDogName(currentName);
      setEditDogBreed(currentBreed);
      setEditDogBreedOther(currentBreedOther);
      setEditDogDob(currentDob);
      setEditDogModal(true);
      
    } catch (e) {
      console.error("Error opening dog edit modal:", e);
      Alert.alert("Error", "Could not open the edit form. Please try again.");
    }
  }, [dogName, dogBreed, dogDob]);

  const saveDogInfo = async () => {
    // Enhanced null checks
    const name = editDogName?.trim() || "";
    const dob = editDogDob?.trim() || "";
    const breedOther = editDogBreedOther?.trim() || "";
    
    let breedToSave = editDogBreed === "Other" ? breedOther : editDogBreed;
    
    if (!name || !breedToSave || !dob) {
      Alert.alert("Missing Info", "All fields are required.");
      return;
    }
    
    if (editDogBreed === "Other" && !breedOther) {
      Alert.alert("Breed Required", "Please specify the breed.");
      return;
    }
    
    if (!isValidDate(dob)) {
      Alert.alert("Invalid Date", "Please enter a valid date in YYYY-MM-DD format.");
      return;
    }
    
    try {
      setDogName(name);
      setDogBreed(breedToSave);
      setDogDob(dob);
      
      await updateDogInfoStorageAndFirestore(familyId, name, breedToSave, dob);
      setEditDogModal(false);
    } catch (e) {
      Alert.alert("Network error", e.message);
    }
  };

  const handleLogout = async () => {
    setFamilyDogLoading(true);
    await AsyncStorage.removeItem("familyId");
    await AsyncStorage.removeItem("dogName");
    await AsyncStorage.removeItem("dogBreed");
    await AsyncStorage.removeItem("dogDob");
    await AsyncStorage.removeItem("completedDailyByDate");
    setFamilyId(null);
    setDogName(null);
    setDogBreed(null);
    setDogDob(null);
    setInputId("");
    setInputDog("");
    setInputBreed("");
    setInputBreedOther("");
    setInputDob("");
    setShowDogInput(false);
    setFamilyDogLoading(false);
  };

  const navigationTabs = [
    { key: "daily", iconLib: Feather, iconName: "calendar" },
    { key: "weekly", iconLib: Feather, iconName: "clock" },
    { key: "progress", iconLib: Feather, iconName: "trending-up" },
    { key: "family", iconLib: Feather, iconName: "users" },
    { key: "notes", iconLib: Feather, iconName: "sticky-note" }
  ];

  // Loading states
  if (familyDogLoading) {
    return (
      <View style={styles.loadingContainer}>
        <ActivityIndicator size="large" color="#2563eb" />
        <Text style={styles.loadingText}>Loading…</Text>
      </View>
    );
  }

  // Setup screens
  if (!familyId || !dogName || !dogBreed || !dogDob) {
    if (!showDogInput) {
      return (
        <View style={styles.setupContainer}>
          <Text style={styles.welcomeTitle}>Welcome!</Text>
          <Text style={styles.setupText}>
            Enter your Family ID to sync training data with your household.
          </Text>
          <Text style={styles.setupSubtext}>
            Use the same Family ID on all your family's devices. Create a new one or enter an existing ID to join.
          </Text>
          <TextInput
            placeholder="Family ID (e.g. ludo-smiths)"
            value={inputId}
            onChangeText={setInputId}
            style={styles.setupInput}
          />
          <TouchableOpacity onPress={handleFamilyIdSubmit} style={styles.setupButton}>
            {checkingFamilyId
              ? <ActivityIndicator size="small" color="#fff"/>
              : <Text style={styles.setupButtonText}>Continue</Text>
            }
          </TouchableOpacity>
        </View>
      );
    } else {
      return (
        <View style={styles.setupContainer}>
          <Text style={styles.welcomeTitle}>Create a New Family Group</Text>
          <Text style={styles.setupText}>
            Set your Family ID and Dog's Details (only needed once).
          </Text>
          <TextInput
            value={inputId}
            editable={false}
            style={[styles.setupInput, styles.disabledInput]}
          />
          <TextInput
            placeholder="Dog Name (e.g. Ludo)"
            value={inputDog}
            onChangeText={setInputDog}
            style={styles.setupInput}
          />
          <RNPickerSelect
            onValueChange={value => setInputBreed(value)}
            value={inputBreed}
            placeholder={{ label: "Select Dog Breed...", value: "" }}
            items={[
              ...commonUKBreeds.map(breed => ({ label: breed, value: breed })),
              { label: "Other", value: "Other" }
            ]}
            style={{
              inputIOS: styles.pickerInput,
              inputAndroid: styles.pickerInput
            }}
          />
          {inputBreed === "Other" && (
            <TextInput
              placeholder="Type breed"
              value={inputBreedOther}
              onChangeText={setInputBreedOther}
              style={styles.setupInput}
            />
          )}
          <TextInput
            placeholder="Date of Birth (YYYY-MM-DD)"
            value={inputDob}
            onChangeText={setInputDob}
            keyboardType={Platform.OS === "ios" ? "numbers-and-punctuation" : "numeric"}
            style={styles.setupInput}
            ref={dobInputRef}
            maxLength={10}
          />
          <TouchableOpacity onPress={handleDogDetailsSubmit} style={styles.setupButton}>
            <Text style={styles.setupButtonText}>Create Family</Text>
          </TouchableOpacity>
        </View>
      );
    }
  }

  if (loading) {
    return (
      <View style={styles.loadingContainer}>
        <ActivityIndicator size="large" color="#2563eb" />
        <Text style={styles.loadingText}>Loading training progress…</Text>
      </View>
    );
  }

  return (
    <SafeAreaView style={styles.container} edges={['top', 'bottom']}>
      <View style={styles.mainContent}>
        <ScrollView contentContainerStyle={styles.scrollContent}>
          {isOffline && (
            <View style={styles.offlineBanner}>
              <Feather name="wifi-off" size={18} color="#fff" />
              <Text style={styles.offlineText}>You are offline. Changes will sync when back online.</Text>
            </View>
          )}
          
          {showDailyConfetti && (
            <>
              <ConfettiCannon key="daily-confetti" count={120} origin={{x: 200, y: 0}} fadeOut autoStart />
              <View style={styles.celebrationBanner}>
                <Feather name="award" size={30} color="#f59e42" />
                <Text style={styles.celebrationText}>Daily Goal Complete! Well done!</Text>
              </View>
            </>
          )}
          
          {showWeeklyConfetti && (
            <>
              <ConfettiCannon key="weekly-confetti" count={200} origin={{x: 100, y: 0}} fadeOut autoStart />
              <View style={styles.celebrationBanner}>
                <Feather name="award" size={30} color="#22c55e" />
                <Text style={styles.celebrationText}>Weekly Goal Achieved! 🎉</Text>
              </View>
            </>
          )}
          
          <View style={styles.header}>
            <View style={styles.headerContent}>
              <View style={styles.headerInfo}>
                <Text style={styles.title}>🐕 {dogName}'s Training Journey</Text>
                <Text style={styles.subtitle}>
                  Family ID: <Text style={styles.familyId}>{familyId}</Text>
                </Text>
                <Text style={styles.subtitle}>
                  {dogBreed} • Week {currentWeek} • {trainingStages[currentStage].name}
                </Text>
                <Text style={styles.subtitle}>
                  Age: <Text style={styles.dogAge}>{currentDogAge}</Text>
                  {"  "}DOB: <Text style={styles.dogDob}>{dogDob}</Text>
                </Text>
              </View>
              <View style={styles.headerStats}>
                <TouchableOpacity onPress={openEditDogModal} style={styles.editButton}>
                  <Feather name="edit-3" size={20} color="#6366f1" />
                </TouchableOpacity>
                <Text style={styles.completionPercent}>
                  {Math.round(completionRate * 100)}%
                </Text>
                <Text style={styles.completionLabel}>This Week</Text>
                <Feather name="users" size={24} color="#a3a3a3" />
              </View>
            </View>
            {syncing && (
              <View style={styles.syncingIndicator}>
                <ActivityIndicator size="small" color="#2563eb" />
                <Text style={styles.syncingText}>Syncing…</Text>
              </View>
            )}
          </View>

          {viewMode === "daily" && (
            <View>
              <View style={[styles.section, styles.progressSection]}>
                <View style={styles.progressHeader}>
                  <Feather name="sun" size={22} color="#f59e42" />
                  <Text style={styles.progressTitle}>Today's Progress</Text>
                </View>
                <View style={styles.progressStats}>
                  <Text style={styles.progressPercent}>
                    {Math.round(dailyRate * 100)}%
                  </Text>
                </View>
              </View>
              
              <View style={styles.section}>
                <Text style={styles.sectionTitle}>
                  <Feather name="sunrise" size={18} /> Today's Training Schedule
                </Text>
                {Object.entries(dailyRoutine).map(([timeSlot, acts]) => (
                  <View key={timeSlot} style={styles.timeSlotContainer}>
                    <Text style={styles.timeSlotTitle}>
                      <Feather name="clock" size={14} />{" "}
                      {timeSlot === "play"
                        ? "Play & Bonding"
                        : `${timeSlot.charAt(0).toUpperCase() + timeSlot.slice(1)} Routine`}
                    </Text>
                    {acts.map((activity, idx) => (
                      <TouchableOpacity
                        key={idx}
                        style={styles.activityRow}
                        onPress={() => toggleDailyActivity(timeSlot, activity)}
                      >
                        {completedToday.includes(`daily-${timeSlot}-${activity}`) ? (
                          <MaterialCommunityIcons name="checkbox-marked-circle" color="#22c55e" size={20} />
                        ) : (
                          <MaterialCommunityIcons name="checkbox-blank-circle-outline" color="#9ca3af" size={20} />
                        )}
                        <Text
                          style={[
                            styles.activityText,
                            completedToday.includes(`daily-${timeSlot}-${activity}`) &&
                              styles.activityCompleted
                          ]}
                        >
                          {activity}
                        </Text>
                      </TouchableOpacity>
                    ))}
                  </View>
                ))}
              </View>
              
              <View style={styles.section}>
                <Text style={styles.sectionTitle}>
                  <Feather name="star" size={18} color="#eab308" /> Week {currentWeek} Focus
                </Text>
                {(weeklyPlans[currentWeek] || []).map((activity, idx) => (
                  <TouchableOpacity
                    key={idx}
                    style={styles.activityRow}
                    onPress={() => toggleActivity(currentWeek, activity)}
                  >
                    {completedActivities.includes(`${currentWeek}-${activity}`) ? (
                      <MaterialCommunityIcons name="checkbox-marked-circle" color="#22c55e" size={22} />
                    ) : (
                      <MaterialCommunityIcons name="checkbox-blank-circle-outline" color="#9ca3af" size={22} />
                    )}
                    <Text
                      style={[
                        styles.activityText,
                        completedActivities.includes(`${currentWeek}-${activity}`) && styles.activityCompleted
                      ]}
                    >
                      {activity}
                    </Text>
                  </TouchableOpacity>
                ))}
              </View>
              
              <View style={styles.section}>
                <Text style={styles.sectionTitle}>
                  <Feather name="bookmark" size={18} /> Daily Notes
                </Text>
                <TextInput
                  style={styles.textInput}
                  multiline
                  placeholder={`How did ${dogName} do today?`}
                  value={dailyNotes[selectedDate] || ""}
                  onChangeText={text => addNote(selectedDate, text)}
                />
              </View>
            </View>
          )}

          {viewMode === "weekly" && (
            <View style={styles.section}>
              <View style={styles.weekNavigation}>
                <TouchableOpacity
                  onPress={() => setCurrentWeek(w => Math.max(1, w - 1))}
                  disabled={currentWeek === 1}
                  style={[styles.weekNavButton, currentWeek === 1 && styles.weekNavDisabled]}
                >
                  <Feather name="chevron-left" size={22} color="#2563eb" />
                  <Text style={styles.weekNav}>Previous Week</Text>
                </TouchableOpacity>
                <Text style={styles.weekTitle}>Week {currentWeek}</Text>
                <TouchableOpacity
                  onPress={() => setCurrentWeek(w => Math.min(MAX_WEEKS, w + 1))}
                  disabled={currentWeek === MAX_WEEKS}
                  style={[styles.weekNavButton, currentWeek === MAX_WEEKS && styles.weekNavDisabled]}
                >
                  <Text style={styles.weekNav}>Next Week</Text>
                  <Feather name="chevron-right" size={22} color="#2563eb" />
                </TouchableOpacity>
              </View>
              
              <View style={styles.weekCard}>
                <Text style={styles.weekCardTitle}>Week {currentWeek}</Text>
                <Text style={styles.weekCardAge}>
                  Age: <Text style={styles.dogAge}>{calcDogAge(dogDob)}</Text>
                </Text>
                {(weeklyPlans[currentWeek] || ["Coming soon..."]).map((activity, idx) => (
                  <TouchableOpacity
                    key={idx}
                    style={styles.activityRow}
                    onPress={() => toggleActivity(currentWeek, activity)}
                  >
                    {completedActivities.includes(`${currentWeek}-${activity}`) ? (
                      <MaterialCommunityIcons name="checkbox-marked-circle" color="#22c55e" size={16} />
                    ) : (
                      <MaterialCommunityIcons name="checkbox-blank-circle-outline" color="#9ca3af" size={16} />
                    )}
                    <Text
                      style={[
                        styles.activityText,
                        completedActivities.includes(`${currentWeek}-${activity}`) && styles.activityCompleted,
                        styles.smallActivityText
                      ]}
                    >
                      {activity}
                    </Text>
                  </TouchableOpacity>
                ))}
              </View>
            </View>
          )}

          {viewMode === "progress" && (
            <View style={styles.section}>
              <Text style={styles.sectionTitle}>
                <Feather name="activity" size={18} color="#22c55e" /> {dogName}'s Progress Journey
              </Text>
              <View style={styles.stagesContainer}>
                {Object.entries(trainingStages).map(([stage, info]) => (
                  <View
                    key={stage}
                    style={[
                      styles.stageBox,
                      { backgroundColor: info.color, opacity: currentStage >= stage ? 1 : 0.5 }
                    ]}
                  >
                    <Text style={styles.stageName}>{info.name}</Text>
                    <Text style={styles.stageRange}>{info.range}</Text>
                  </View>
                ))}
              </View>
              
              <Text style={styles.milestonesTitle}>Key Milestones</Text>
              {[
                { week: 1, milestone: "House training established", completed: currentWeek >= 4 },
                { week: 6, milestone: "Basic commands mastered", completed: currentWeek >= 8 },
                { week: 12, milestone: "Impulse control developed", completed: currentWeek >= 16 },
                { week: 20, milestone: "Advanced obedience achieved", completed: currentWeek >= 26 },
                { week: 30, milestone: "Adolescent challenges managed", completed: currentWeek >= 35 }
              ].map((item, idx) => (
                <TouchableOpacity
                  key={idx}
                  style={styles.milestoneRow}
                  onPress={() => toggleActivity("milestone", item.milestone)}
                >
                  {completedActivities.includes(`milestone-${item.milestone}`) ? (
                    <MaterialCommunityIcons name="checkbox-marked-circle" color="#22c55e" size={20} />
                  ) : (
                    <MaterialCommunityIcons name="checkbox-blank-circle-outline" color="#9ca3af" size={20} />
                  )}
                  <Text
                    style={[
                      styles.activityText,
                      completedActivities.includes(`milestone-${item.milestone}`) && styles.activityCompleted
                    ]}
                  >
                    {item.milestone}
                  </Text>
                  <Text style={styles.milestoneWeek}>Week {item.week}</Text>
                </TouchableOpacity>
              ))}
            </View>
          )}

          {viewMode === "family" && (
            <View style={styles.section}>
              <View style={styles.identitySelector}>
                <Text style={styles.identityTitle}>Set Your Identity</Text>
                <Text style={styles.identitySubtitle}>
                  Select who you are on this device to edit your notes:
                </Text>
                <View style={styles.identityButtons}>
                  {family.map(member => (
                    <TouchableOpacity
                      key={member.id}
                      onPress={() => {
                        setCurrentUserName(member.name);
                        AsyncStorage.setItem("memberName", member.name);
                        Alert.alert("Identity Set", `You are now identified as ${member.name} on this device.`);
                      }}
                      style={[
                        styles.identityButton,
                        currentUserName === member.name && styles.identityButtonActive
                      ]}
                    >
                      <Text style={[
                        styles.identityButtonText,
                        currentUserName === member.name && styles.identityButtonTextActive
                      ]}>
                        {member.name}
                      </Text>
                    </TouchableOpacity>
                  ))}
                </View>
              </View>

              <Text style={styles.sectionTitle}>
                <Feather name="users" size={18} color="#a78bfa" /> Family Collaboration
              </Text>
              
              <View>
                <Text style={styles.familyTitle}>Training Team</Text>
                {family.map((member, idx) => (
                  <View key={member.id} style={styles.familyRow}>
                    <Ionicons name="person-circle" size={28} color="#2563eb" />
                    {editingMemberId === member.id ? (
                      <TextInput
                        style={styles.familyEditInput}
                        value={editFamilyName[member.id]}
                        onChangeText={text =>
                          setEditFamilyName(n => ({ ...n, [member.id]: text }))
                        }
                        autoFocus
                        onSubmitEditing={() => handleSaveFamilyMember(member.id, editFamilyName[member.id])}
                        onBlur={() => handleSaveFamilyMember(member.id, editFamilyName[member.id])}
                      />
                    ) : (
                      <TouchableOpacity style={styles.familyNameContainer} onLongPress={() => handleEditFamilyMember(member.id)}>
                        <Text style={styles.familyName}>{member.name}</Text>
                      </TouchableOpacity>
                    )}
                    <Text style={styles.familyStatus}>Active</Text>
                    <TouchableOpacity onPress={() => handleEditFamilyMember(member.id)} style={styles.familyEditButton}>
                      <Feather name="edit" size={18} color="#888" />
                    </TouchableOpacity>
                    <TouchableOpacity onPress={() => handleRemoveFamilyMember(member.id)} style={styles.familyDeleteButton}>
                      <Feather name="trash-2" size={18} color="#dc2626" />
                    </TouchableOpacity>
                  </View>
                ))}
                
                <View style={styles.addFamilyContainer}>
                  <TextInput
                    style={styles.addFamilyInput}
                    placeholder="Add new member..."
                    value={newFamilyName}
                    onChangeText={setNewFamilyName}
                    onSubmitEditing={handleAddFamilyMember}
                  />
                  <TouchableOpacity onPress={handleAddFamilyMember} style={styles.addFamilyButton}>
                    <Feather name="plus" size={20} color="#fff" />
                  </TouchableOpacity>
                </View>
              </View>
              
              <View style={styles.syncInfo}>
                <View style={styles.syncDot} />
                <Text style={styles.syncText}>All data synced across devices</Text>
              </View>
              <Text style={styles.syncSubtext}>
                Training progress, notes, and schedules are automatically shared with all family members.
              </Text>
              
              <View style={styles.familyActions}>
                <TouchableOpacity style={styles.editDogButton} onPress={openEditDogModal}>
                  <Feather name="edit-3" size={16} color="#6366f1" />
                  <Text style={styles.editDogText}>Edit Dog Info</Text>
                </TouchableOpacity>
                
                <TouchableOpacity
                  style={styles.logoutButton}
                  onPress={handleLogout}
                >
                  <Feather name="log-out" size={16} color="#fff" />
                  <Text style={styles.logoutText}>Log Out</Text>
                </TouchableOpacity>
              </View>
            </View>
          )}

          {viewMode === "notes" && (
            <View style={styles.section}>
              <Text style={styles.sectionTitle}>
                <Feather name="sticky-note" size={18} color="#2563eb" /> Shared Notes
              </Text>
              
              <View style={styles.addNoteContainer}>
                <TextInput
                  style={styles.addNoteInput}
                  placeholder="Add a note for your family…"
                  value={newSharedNote}
                  onChangeText={setNewSharedNote}
                  onSubmitEditing={handleAddSharedNote}
                  returnKeyType="done"
                />
                <TouchableOpacity
                  onPress={handleAddSharedNote}
                  style={[styles.addNoteButton, !newSharedNote.trim() && styles.addNoteButtonDisabled]}
                  disabled={!newSharedNote.trim()}
                >
                  <Text style={styles.addNoteButtonText}>Add Note</Text>
                </TouchableOpacity>
              </View>
              
              {notesLoading ? (
                <ActivityIndicator size="small" color="#2563eb" />
              ) : sharedNotes && sharedNotes.length > 0 ? (
                sharedNotes.map(note => (
                  <View key={note.id} style={styles.noteCard}>
                    {editingNoteId === note.id ? (
                      <>
                        <TextInput
                          style={styles.textInput}
                          multiline
                          value={editingNoteText}
                          onChangeText={setEditingNoteText}
                          autoFocus
                        />
                        <View style={styles.noteEditActions}>
                          <TouchableOpacity 
                            onPress={() => setEditingNoteId(null)} 
                            style={styles.noteCancelButton}
                          >
                            <Text style={styles.noteCancelText}>Cancel</Text>
                          </TouchableOpacity>
                          <TouchableOpacity 
                            onPress={() => saveNoteEdit(note.id)}
                            style={styles.noteSaveButton}
                          >
                            <Text style={styles.noteSaveText}>Save</Text>
                          </TouchableOpacity>
                        </View>
                      </>
                    ) : (
                      <>
                        <Text style={styles.noteText}>{note.text}</Text>
                        <View style={styles.noteFooter}>
                          <Text style={styles.noteAuthor}>By {note.author}</Text>
                          <View style={styles.noteActions}>
                            {note.author === currentUserName && (
                              <TouchableOpacity 
                                onPress={() => {
                                  setEditingNoteId(note.id);
                                  setEditingNoteText(note.text);
                                }}
                                style={styles.noteEditButton}
                              >
                                <Feather name="edit-2" size={16} color="#6366f1" />
                              </TouchableOpacity>
                            )}
                            <Text style={styles.noteTimestamp}>
                              {note.timestamp ? new Date(note.timestamp).toLocaleString() : ""}
                              {note.edited && " (edited)"}
                            </Text>
                          </View>
                        </View>
                      </>
                    )}
                  </View>
                ))
              ) : (
                <Text style={styles.noNotesText}>No notes yet. Be the first to add one!</Text>
              )}
              
              <View style={styles.tipsBox}>
                <Text style={styles.tipsTitle}>
                  <Feather name="book" size={18} color="#fff" /> {dogName}'s Training Tips
                </Text>
                <Text style={styles.tip}>
                  <Text style={styles.tipLabel}>Mental Stimulation: </Text>
                  {dogBreed || "Border Collies"} need mental challenges. Use puzzle toys and training games daily.
                </Text>
                <Text style={styles.tip}>
                  <Text style={styles.tipLabel}>Positive Reinforcement: </Text>
                  Use treats, praise, and play. End sessions on a successful note.
                </Text>
                <Text style={styles.tip}>
                  <Text style={styles.tipLabel}>Consistency: </Text>
                  Use the same commands and routines. A tired brain is better than a tired body!
                </Text>
                                {breedTips[dogBreed] && (
                  <>
                    <Text style={styles.breedTipsTitle}>
                      Tips for {dogBreed}:
                    </Text>
                    {breedTips[dogBreed].map((tip, idx) => (
                      <Text key={idx} style={styles.tip}>
                        {tip}
                      </Text>
                    ))}
                  </>
                )}
              </View>
            </View>
          )}

          {/* App version footer */}
          <View style={styles.appFooter}>
            <Text style={styles.appVersion}>v{APP_VERSION} • Built {BUILD_DATE}</Text>
            <Text style={styles.currentDateTime}>Current: {CURRENT_DATE} {CURRENT_TIME}</Text>
          </View>
        </ScrollView>
        
        <View style={styles.tabBar}>
          {navigationTabs.map(tab => (
            <TouchableOpacity
              key={tab.key}
              style={[styles.tabButton, viewMode === tab.key && styles.tabActive]}
              onPress={() => setViewMode(tab.key)}
            >
              <tab.iconLib
                name={tab.iconName}
                size={22}
                color={viewMode === tab.key ? "#2563eb" : "#64748b"}
              />
              <Text
                style={{
                  fontSize: 12,
                  marginTop: 2,
                  color: viewMode === tab.key ? "#2563eb" : "#64748b"
                }}
              >
                {tab.key.charAt(0).toUpperCase() + tab.key.slice(1)}
              </Text>
            </TouchableOpacity>
          ))}
        </View>
        
        <Modal visible={editDogModal} transparent animationType="slide">
          <View style={styles.centeredView}>
            <View style={styles.modalView}>
              <Text style={styles.modalTitle}>
                Edit {dogName}'s Information
              </Text>
              <TextInput
                placeholder="Dog Name"
                value={editDogName}
                onChangeText={setEditDogName}
                style={styles.modalInput}
              />
              <RNPickerSelect
                onValueChange={value => setEditDogBreed(value)}
                value={editDogBreed}
                placeholder={{ label: "Select Dog Breed...", value: "" }}
                items={[
                  ...commonUKBreeds.map(breed => ({ label: breed, value: breed })),
                  { label: "Other", value: "Other" }
                ]}
                style={{
                  inputIOS: styles.modalPickerInput,
                  inputAndroid: styles.modalPickerInput
                }}
              />
              {editDogBreed === "Other" && (
                <TextInput
                  placeholder="Type breed"
                  value={editDogBreedOther}
                  onChangeText={setEditDogBreedOther}
                  style={styles.modalInput}
                />
              )}
              <TextInput
                placeholder="Date of Birth (YYYY-MM-DD)"
                value={editDogDob}
                onChangeText={setEditDogDob}
                keyboardType={Platform.OS === "ios" ? "numbers-and-punctuation" : "numeric"}
                style={styles.modalInput}
                maxLength={10}
              />
              <View style={styles.modalActions}>
                <TouchableOpacity
                  style={styles.modalCancelButton}
                  onPress={() => setEditDogModal(false)}
                >
                  <Text style={styles.modalCancelText}>Cancel</Text>
                </TouchableOpacity>
                <TouchableOpacity 
                  style={styles.modalSaveButton}
                  onPress={saveDogInfo}
                >
                  <Text style={styles.modalSaveText}>Save Changes</Text>
                </TouchableOpacity>
              </View>
            </View>
          </View>
        </Modal>
      </View>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  // Error boundary styles
  errorContainer: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
    padding: 20,
    backgroundColor: "#f9fafb",
  },
  errorTitle: {
    fontSize: 20,
    fontWeight: "bold",
    marginTop: 16,
    marginBottom: 8,
    color: "#ef4444",
  },
  errorText: {
    fontSize: 16,
    color: "#6b7280",
    textAlign: "center",
    marginBottom: 20,
  },
  errorButton: {
    backgroundColor: "#2563eb",
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 6,
  },
  errorButtonText: {
    color: "#fff",
    fontWeight: "bold",
  },

  // Loading and setup screens
  loadingContainer: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
  },
  loadingText: {
    marginTop: 16,
  },
  setupContainer: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
    padding: 32,
  },
  welcomeTitle: {
    fontSize: 20,
    fontWeight: "bold",
    marginBottom: 24,
  },
  setupText: {
    fontSize: 16,
    marginBottom: 8,
    textAlign: "center",
  },
  setupSubtext: {
    fontSize: 14,
    marginBottom: 16,
    color: "#555",
    textAlign: "center",
  },
  setupInput: {
    borderWidth: 1,
    borderColor: "#ccc",
    padding: 10,
    marginBottom: 12,
    borderRadius: 6,
    width: "100%",
  },
  disabledInput: {
    backgroundColor: "#eee",
  },
  setupButton: {
    backgroundColor: "#2563eb",
    padding: 12,
    borderRadius: 6,
    width: "100%",
    alignItems: "center",
  },
  setupButtonText: {
    color: "#fff",
    fontWeight: "bold",
  },
  pickerInput: {
    borderWidth: 1,
    borderColor: "#ccc",
    padding: 10,
    marginBottom: 12,
    borderRadius: 6,
    width: "100%",
    fontSize: 16,
  },

  // Main app styles
  container: {
    flex: 1,
    backgroundColor: "#f3f4f6",
  },
  mainContent: {
    flex: 1,
  },
  scrollContent: {
    padding: 10,
    paddingBottom: 100,
  },

  // Banners
  offlineBanner: {
    backgroundColor: "#ef4444",
    padding: 8,
    flexDirection: "row",
    alignItems: "center",
    borderRadius: 6,
    marginBottom: 8,
  },
  offlineText: {
    color: "#fff",
    fontWeight: "500",
    marginLeft: 6,
  },
  celebrationBanner: {
    backgroundColor: "#fef3c7",
    padding: 12,
    flexDirection: "row",
    alignItems: "center",
    borderRadius: 6,
    marginBottom: 8,
    justifyContent: "center",
    borderWidth: 1,
    borderColor: "#f59e0b",
  },
  celebrationText: {
    color: "#92400e",
    fontWeight: "bold",
    fontSize: 16,
    marginLeft: 6,
  },

  // Header with LIGHT GREEN background
  header: {
    marginBottom: 16,
    padding: 10,
    backgroundColor: "#bbf7d0", // Light green background
    borderRadius: 8,
    elevation: 1,
  },
  headerContent: {
    flexDirection: "row",
    justifyContent: "space-between",
    alignItems: "center",
  },
  headerInfo: {
    flex: 1,
  },
  title: {
    fontSize: 20,
    fontWeight: "bold",
    marginBottom: 2,
  },
  subtitle: {
    fontSize: 14,
    color: "#6b7280",
    marginBottom: 2,
  },
  familyId: {
    color: "#2563eb",
  },
  dogAge: {
    color: "#16a34a",
    fontWeight: "bold",
  },
  dogDob: {
    color: "#555",
  },
  headerStats: {
    alignItems: "center",
  },
  editButton: {
    marginBottom: 2,
  },
  completionPercent: {
    color: "#2563eb",
    fontWeight: "bold",
    fontSize: 24,
  },
  completionLabel: {
    color: "#6b7280",
    fontSize: 12,
  },
  syncingIndicator: {
    flexDirection: "row",
    alignItems: "center",
    marginTop: 8,
  },
  syncingText: {
    marginLeft: 6,
    color: "#2563eb",
  },

  // Sections
  section: {
    backgroundColor: "#fff",
    borderRadius: 8,
    padding: 16,
    marginBottom: 16,
    elevation: 1,
  },
  sectionTitle: {
    fontSize: 16,
    fontWeight: "bold",
    marginBottom: 10,
  },
  progressSection: {
    marginBottom: 8,
    flexDirection: "row",
    alignItems: "center",
    justifyContent: "space-between",
  },
  progressHeader: {
    flexDirection: "row",
    alignItems: "center",
  },
  progressTitle: {
    marginLeft: 8,
    fontWeight: "bold",
    fontSize: 16,
  },
  progressStats: {
    alignItems: "center",
  },
  progressPercent: {
    color: "#f59e42",
    fontWeight: "bold",
    fontSize: 22,
  },

  // Time slots and activities
  timeSlotContainer: {
    marginBottom: 15,
  },
  timeSlotTitle: {
    fontWeight: "600",
    marginBottom: 5,
  },
  activityRow: {
    flexDirection: "row",
    alignItems: "center",
    paddingVertical: 6,
  },
  activityText: {
    marginLeft: 8,
    fontSize: 15,
  },
  smallActivityText: {
    fontSize: 14,
  },
  activityCompleted: {
    color: "#22c55e",
    textDecorationLine: "line-through",
    fontStyle: "italic",
  },

  // Text input
  textInput: {
    borderWidth: 1,
    borderColor: "#d1d5db",
    borderRadius: 6,
    padding: 10,
    minHeight: 80,
    backgroundColor: "#f9fafb",
  },

  // Week navigation
  weekNavigation: {
    flexDirection: "row",
    alignItems: "center",
    justifyContent: "space-between",
    marginBottom: 10,
  },
  weekNavButton: {
    flexDirection: "row",
    alignItems: "center",
  },
  weekNavDisabled: {
    opacity: 0.3,
  },
  weekNav: {
    color: "#2563eb",
    fontWeight: "500",
  },
  weekTitle: {
    fontWeight: "bold",
    fontSize: 18,
    color: "#2563eb",
  },
  weekCard: {
    borderWidth: 1,
    borderColor: "#2563eb",
    backgroundColor: "#dbeafe",
    borderRadius: 8,
    padding: 12,
  },
  weekCardTitle: {
    fontWeight: "bold",
  },
  weekCardAge: {
    fontSize: 14,
    marginBottom: 5,
  },

  // Progress stages
  stagesContainer: {
    flexDirection: "row",
    flexWrap: "wrap",
    marginBottom: 10,
  },
  stageBox: {
    padding: 8,
    borderRadius: 6,
    margin: 4,
    minWidth: 100,
  },
  stageName: {
    fontWeight: "600",
  },
  stageRange: {
    fontSize: 12,
  },
  milestonesTitle: {
    fontWeight: "bold",
    marginBottom: 8,
  },
  milestoneRow: {
    flexDirection: "row",
    alignItems: "center",
    paddingVertical: 6,
  },
  milestoneWeek: {
    marginLeft: "auto",
    color: "#6b7280",
    fontSize: 12,
  },

  // Family section
  identitySelector: {
    marginBottom: 16,
    backgroundColor: "#f0f9ff",
    padding: 12,
    borderRadius: 8,
  },
  identityTitle: {
    fontSize: 16,
    fontWeight: "bold",
    marginBottom: 4,
  },
  identitySubtitle: {
    fontSize: 14,
    color: "#4b5563",
    marginBottom: 8,
  },
  identityButtons: {
    flexDirection: "row",
    flexWrap: "wrap",
  },
  identityButton: {
    backgroundColor: "#e5e7eb",
    padding: 8,
    borderRadius: 6,
    marginRight: 8,
    marginBottom: 8,
  },
  identityButtonActive: {
    backgroundColor: "#2563eb",
  },
  identityButtonText: {
    color: "#374151",
  },
  identityButtonTextActive: {
    color: "#fff",
  },
  familyTitle: {
    fontWeight: "bold",
    marginBottom: 8,
  },
  familyRow: {
    flexDirection: "row",
    alignItems: "center",
    paddingVertical: 8,
    borderBottomWidth: 1,
    borderBottomColor: "#f1f5f9",
  },
  familyEditInput: {
    flex: 1,
    marginLeft: 6,
    marginRight: 6,
    height: 36,
    minHeight: 0,
    paddingVertical: 3,
    fontSize: 16,
    borderWidth: 1,
    borderColor: "#d1d5db",
    borderRadius: 6,
    padding: 10,
    backgroundColor: "#f9fafb",
  },
  familyNameContainer: {
    flex: 1,
    marginLeft: 6,
  },
  familyName: {
    fontSize: 16,
  },
  familyStatus: {
    marginLeft: 8,
    color: "#22c55e",
  },
  familyEditButton: {
    marginLeft: 6,
  },
  familyDeleteButton: {
    marginLeft: 4,
  },
  addFamilyContainer: {
    flexDirection: "row",
    alignItems: "center",
    marginTop: 6,
  },
  addFamilyInput: {
    flex: 1,
    marginRight: 6,
    height: 36,
    minHeight: 0,
    paddingVertical: 3,
    fontSize: 16,
    borderWidth: 1,
    borderColor: "#d1d5db",
    borderRadius: 6,
    padding: 10,
    backgroundColor: "#f9fafb",
  },
  addFamilyButton: {
    padding: 6,
    backgroundColor: "#2563eb",
    borderRadius: 6,
  },
  syncInfo: {
    marginTop: 18,
    flexDirection: "row",
    alignItems: "center",
  },
  syncDot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: "#22c55e",
  },
  syncText: {
    color: "#22c55e",
    marginLeft: 4,
  },
  syncSubtext: {
    color: "#6b7280",
    fontSize: 12,
    marginTop: 2,
  },
  familyActions: {
    marginTop: 18,
    alignSelf: "flex-end",
  },
  editDogButton: {
    flexDirection: "row",
    alignItems: "center",
    marginBottom: 8,
  },
  editDogText: {
    color: "#6366f1",
    marginLeft: 4,
  },
  logoutButton: {
    flexDirection: "row",
    alignItems: "center",
    backgroundColor: "#ef4444",
    borderRadius: 8,
    paddingVertical: 8,
    paddingHorizontal: 14,
    elevation: 2,
  },
  logoutText: {
    color: "#fff",
    fontWeight: "bold",
    marginLeft: 8,
  },

  // Notes section
  addNoteContainer: {
    marginBottom: 14,
  },
  addNoteInput: {
    minHeight: 44,
    paddingVertical: 10,
    marginBottom: 8,
    borderWidth: 1,
    borderColor: "#d1d5db",
    borderRadius: 6,
    padding: 10,
    backgroundColor: "#f9fafb",
  },
  addNoteButton: {
    backgroundColor: "#2563eb",
    alignSelf: "flex-end",
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 6,
  },
  addNoteButtonDisabled: {
    opacity: 0.5,
  },
  addNoteButtonText: {
    color: "#fff",
    fontWeight: "bold",
  },
  noteCard: {
    backgroundColor: "#e0e7ff",
    borderRadius: 8,
    padding: 10,
    marginBottom: 10,
  },
  noteEditActions: {
    flexDirection: "row",
    justifyContent: "flex-end",
    marginTop: 8,
  },
  noteCancelButton: {
    marginRight: 12,
    padding: 6,
  },
  noteCancelText: {
    color: "#6b7280",
  },
  noteSaveButton: {
    backgroundColor: "#2563eb",
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 4,
  },
  noteSaveText: {
    color: "#fff",
  },
  noteText: {
    fontSize: 15,
    color: "#1e293b",
  },
  noteFooter: {
    flexDirection: "row",
    justifyContent: "space-between",
    marginTop: 5,
    alignItems: "center",
  },
  noteAuthor: {
    fontSize: 12,
    color: "#6366f1",
  },
  noteActions: {
    flexDirection: "row",
    alignItems: "center",
  },
  noteEditButton: {
    marginRight: 8,
  },
  noteTimestamp: {
    fontSize: 11,
    color: "#64748b",
  },
  noNotesText: {
    color: "#6b7280",
    fontSize: 14,
  },

  // Tips box
  tipsBox: {
    backgroundColor: "#2563eb",
    borderRadius: 8,
    padding: 12,
    marginTop: 20,
  },
  tipsTitle: {
    fontWeight: "bold",
    color: "#fff",
    fontSize: 16,
    marginBottom: 8,
  },
  tip: {
    color: "#fff",
    marginBottom: 8,
  },
  tipLabel: {
    fontWeight: "bold",
  },
  breedTipsTitle: {
    color: "#fff",
    fontWeight: "bold",
    marginTop: 10,
  },

  // App footer
  appFooter: {
    alignItems: "center",
    marginTop: 20,
    marginBottom: 10,
  },
  appVersion: {
    color: "#9ca3af",
    fontSize: 12,
  },
  currentDateTime: {
    color: "#9ca3af",
    fontSize: 10,
    marginTop: 2,
  },

  // Tab bar
  tabBar: {
    flexDirection: "row",
    backgroundColor: "#fff",
    position: "absolute",
    bottom: 0,
    left: 0,
    right: 0,
    borderTopWidth: 1,
    borderTopColor: "#e5e7eb",
    paddingVertical: 6,
  },
  tabButton: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
    paddingVertical: 8,
  },
  tabActive: {
    backgroundColor: "#f1f5f9",
    borderRadius: 8,
  },

  // Modal
  centeredView: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
    backgroundColor: "rgba(0, 0, 0, 0.5)",
  },
  modalView: {
    backgroundColor: "#fff",
    borderRadius: 12,
    padding: 22,
    width: "85%",
    alignItems: "center",
    shadowColor: "#000",
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
    elevation: 5,
  },
  modalTitle: {
    fontWeight: "bold",
    fontSize: 18,
    marginBottom: 16,
  },
  modalInput: {
    borderWidth: 1,
    borderColor: "#d1d5db",
    borderRadius: 6,
    padding: 10,
    width: "100%",
    marginBottom: 12,
  },
  modalPickerInput: {
    borderWidth: 1,
    borderColor: "#d1d5db",
    padding: 10,
    marginBottom: 12,
    borderRadius: 6,
    width: "100%",
    fontSize: 16,
  },
  modalActions: {
    flexDirection: "row",
    justifyContent: "space-between",
    width: "100%",
  },
  modalCancelButton: {
    paddingVertical: 10,
    paddingHorizontal: 16,
    borderRadius: 6,
    borderWidth: 1,
    borderColor: "#d1d5db",
  },
  modalCancelText: {
    color: "#6b7280",
  },
  modalSaveButton: {
    backgroundColor: "#2563eb",
    paddingVertical: 10,
    paddingHorizontal: 16,
    borderRadius: 6,
  },
  modalSaveText: {
    color: "#fff",
    fontWeight: "bold",
  },
});

// Wrap the app with error boundary
export default function App() {
  return (
    <ErrorBoundary>
      <SafeAreaProvider>
        <MainApp />
      </SafeAreaProvider>
    </ErrorBoundary>
  );
}
                