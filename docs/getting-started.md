// RideBoostApp.js (React Native + Firebase)

import React, { useState, useEffect } from 'react';
import {
  View, Text, FlatList, Image, TouchableOpacity,
  TextInput, Button, StyleSheet, ActivityIndicator,
} from 'react-native';
import auth from '@react-native-firebase/auth';
import firestore from '@react-native-firebase/firestore';
import storage from '@react-native-firebase/storage';
import * as ImagePicker from 'expo-image-picker';
import MapView, { Marker } from 'react-native-maps';

export default function RideBoostApp() {
  const [user, setUser] = useState(null);
  const [challenges, setChallenges] = useState([]);
  const [posts, setPosts] = useState([]);
  const [selectedChallenge, setSelectedChallenge] = useState(null);
  const [uploading, setUploading] = useState(false);
  const [image, setImage] = useState(null);
  const [location, setLocation] = useState(null);

  // On mount, set auth listener
  useEffect(() => {
    const unsubscribe = auth().onAuthStateChanged(u => {
      setUser(u);
      if (u) loadChallenges();
    });
    return unsubscribe;
  }, []);

  // Load sample challenges
  function loadChallenges() {
    firestore().collection('challenges')
      .onSnapshot(snapshot => {
        setChallenges(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
      });
    firestore().collection('posts').orderBy('createdAt', 'desc').limit(20)
      .onSnapshot(snapshot => {
        setPosts(snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() })));
      });
  }

  // Sign in anonymously for demo
  async function signIn() {
    try {
      await auth().signInAnonymously();
    } catch (e) {
      alert('Sign in error: ' + e.message);
    }
  }

  // Pick photo or video
  async function pickMedia() {
    let result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.All,
      quality: 0.7,
    });
    if (!result.cancelled) {
      setImage(result.uri);
    }
  }

  // Upload media and create post
  async function submitPost() {
    if (!image || !selectedChallenge) return alert('Select challenge & media');

    setUploading(true);
    try {
      const filename = image.split('/').pop();
      const storageRef = storage().ref(`posts/${user.uid}/${filename}`);

      await storageRef.putFile(image);
      const url = await storageRef.getDownloadURL();

      await firestore().collection('posts').add({
        userId: user.uid,
        challengeId: selectedChallenge.id,
        mediaUrl: url,
        createdAt: firestore.FieldValue.serverTimestamp(),
        xpEarned: selectedChallenge.xp || 10,
      });

      // Update user XP
      const userRef = firestore().collection('users').doc(user.uid);
      await userRef.set({ xp: firestore.FieldValue.increment(selectedChallenge.xp || 10) }, { merge: true });

      setImage(null);
      setSelectedChallenge(null);
      alert('Posted with RideBoost effect!');

    } catch (e) {
      alert('Upload error: ' + e.message);
    }
    setUploading(false);
  }

  // Render glow overlay on image (simple)
  function RideBoostEffect({ uri }) {
    return (
      <View style={{ position: 'relative' }}>
        <Image source={{ uri }} style={{ width: 300, height: 200, borderRadius: 10 }} />
        <View style={styles.glowOverlay} />
      </View>
    );
  }

  if (!user) {
    return (
      <View style={styles.centered}>
        <Text style={{ fontSize: 20, marginBottom: 20 }}>Welcome to RideBoost!</Text>
        <Button title="Start Riding (Sign In)" onPress={signIn} />
      </View>
    );
  }

  return (
    <View style={styles.container}>

      <Text style={styles.header}>RideBoost Challenges</Text>

      <FlatList
        data={challenges}
        keyExtractor={item => item.id}
        horizontal
        showsHorizontalScrollIndicator={false}
        style={{ maxHeight: 120 }}
        renderItem={({ item }) => (
          <TouchableOpacity
            onPress={() => setSelectedChallenge(item)}
            style={[
              styles.challengeCard,
              selectedChallenge?.id === item.id && styles.challengeCardSelected,
            ]}
          >
            <Text style={{ fontWeight: 'bold' }}>{item.title}</Text>
            <Text style={{ color: '#0ff' }}>{item.xp} XP</Text>
          </TouchableOpacity>
        )}
      />

      <View style={{ marginTop: 10 }}>
        <Button title="Pick Photo/Video to Post" onPress={pickMedia} />
        {image && <RideBoostEffect uri={image} />}
      </View>

      <Button
        title={uploading ? 'Posting...' : 'Post RideBoost'}
        onPress={submitPost}
        disabled={uploading}
      />

      <Text style={[styles.header, { marginTop: 20 }]}>Recent RideBoost Posts</Text>

      <FlatList
        data={posts}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <View style={styles.postCard}>
            <Text style={{ fontWeight: 'bold' }}>Challenge: {item.challengeId}</Text>
            <Image source={{ uri: item.mediaUrl }} style={{ width: '100%', height: 180, borderRadius: 10 }} />
            <Text>XP Earned: {item.xpEarned}</Text>
          </View>
        )}
      />

      {/* TODO: Add Map, Friends, Leaderboard, Store, etc */}

    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#121212' },
  header: { fontSize: 24, fontWeight: 'bold', color: '#0ff' },
  challengeCard: {
    backgroundColor: '#222',
    padding: 16,
    marginHorizontal: 8,
    borderRadius: 10,
  },
  challengeCardSelected: {
    borderWidth: 2,
    borderColor: '#0ff',
  },
  postCard: {
    backgroundColor: '#222',
    marginVertical: 10,
    padding: 10,
    borderRadius: 10,
  },
  glowOverlay: {
    ...StyleSheet.absoluteFillObject,
    borderRadius: 10,
    borderColor: '#0ff',
    borderWidth: 4,
    opacity: 0.6,
    shadowColor: '#0ff',
    shadowRadius: 20,
    shadowOpacity: 0.8,
  },
  centered: { flex: 1, justifyContent: 'center', alignItems: 'center' },
});

