# integration-mobile-companion

> **Priority**: P2 (Enhancement)
> **Status**: Draft
> **Module**: `amplifier-mobile`

## Overview

Mobile companion app (iOS/Android) for Amplifier. Quick queries on the go, notification handling for CI/CD events, code review approvals, and session continuity between desktop and mobile.

### Value Proposition

| Without | With |
|---------|------|
| Must be at desk for queries | Quick answers anywhere |
| Miss CI alerts until back | Real-time notifications |
| PR reviews wait for desktop | Approve/comment from phone |
| Context lost between devices | Seamless session continuity |

---

## Features

### 1. Quick Query Interface

Fast, voice-enabled queries about your codebase.

```
┌─────────────────────────────────────────┐
│ ← Amplifier                        ⚙️   │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ 🎤 "How does auth work?"           ││
│  └─────────────────────────────────────┘│
│                                         │
│  Recent Queries                         │
│  ─────────────────────────────────────  │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ How does payment retry work?        ││
│  │ 2 hours ago • myproject            ││
│  └─────────────────────────────────────┘│
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ What tests cover UserService?       ││
│  │ Yesterday • myproject              ││
│  └─────────────────────────────────────┘│
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ Explain the caching strategy        ││
│  │ 2 days ago • other-project         ││
│  └─────────────────────────────────────┘│
│                                         │
│                                         │
│  [Projects ▼]           [🎤 Ask]       │
│                                         │
└─────────────────────────────────────────┘
```

### 2. Query Results View

Mobile-optimized response display.

```
┌─────────────────────────────────────────┐
│ ← Query Results                    📋   │
├─────────────────────────────────────────┤
│                                         │
│  How does authentication work?          │
│  myproject • Just now                   │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  The auth system uses JWT tokens:       │
│                                         │
│  **1. Login Flow**                      │
│  User credentials validated against     │
│  database, JWT issued with 24h expiry.  │
│                                         │
│  📄 src/auth/login.ts:23               │
│                                         │
│  **2. Token Validation**                │
│  Middleware extracts and validates      │
│  token on each request.                 │
│                                         │
│  📄 src/auth/middleware.ts:45          │
│                                         │
│  **3. Refresh Logic**                   │
│  Tokens refreshed automatically before  │
│  expiration via refresh endpoint.       │
│                                         │
│  📄 src/auth/refresh.ts:12             │
│                                         │
│  [View on Desktop] [Share] [Copy]       │
│                                         │
└─────────────────────────────────────────┘
```

### 3. Notification Center

CI/CD alerts, PR updates, and system notifications.

```
┌─────────────────────────────────────────┐
│ ← Notifications                   Clear │
├─────────────────────────────────────────┤
│                                         │
│  Today                                  │
│  ─────────────────────────────────────  │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ ❌ Build Failed                     ││
│  │ myproject • main • 10 min ago       ││
│  │                                      ││
│  │ Test suite failed: 3 failures       ││
│  │ [View Details] [Ask Amplifier]      ││
│  └─────────────────────────────────────┘│
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ 🔔 Review Requested                 ││
│  │ PR #234 • @alice • 30 min ago       ││
│  │                                      ││
│  │ "Add payment retry logic"           ││
│  │ [Quick Review] [Open PR]            ││
│  └─────────────────────────────────────┘│
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ ✅ Deploy Complete                  ││
│  │ myproject • staging • 1 hour ago    ││
│  │                                      ││
│  │ v2.3.1 deployed successfully        ││
│  │ [View Logs]                         ││
│  └─────────────────────────────────────┘│
│                                         │
│  Yesterday                              │
│  ─────────────────────────────────────  │
│  ...                                    │
│                                         │
└─────────────────────────────────────────┘
```

### 4. Quick PR Review

Mobile-optimized PR review interface.

```
┌─────────────────────────────────────────┐
│ ← PR #234                         🔗    │
├─────────────────────────────────────────┤
│                                         │
│  Add payment retry logic                │
│  @alice → main • +142 -23              │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  🤖 Amplifier Summary                   │
│  ─────────────────────────────────────  │
│  This PR adds exponential backoff       │
│  retry logic for Stripe API calls.      │
│                                         │
│  **Key Changes:**                       │
│  • New RetryStrategy class              │
│  • 3 retries with 1s base delay         │
│  • Jitter to prevent thundering herd    │
│                                         │
│  **Concerns:**                          │
│  ⚠️ Missing test for max retry case    │
│  ⚠️ Consider adding circuit breaker    │
│                                         │
│  ─────────────────────────────────────  │
│                                         │
│  Files (3)                              │
│  ├── src/payments/processor.ts  +45    │
│  ├── src/payments/retry.ts      +89    │
│  └── tests/payments/retry.test.ts +8   │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ [✅ Approve] [💬 Comment] [❌ Deny] ││
│  └─────────────────────────────────────┘│
│                                         │
└─────────────────────────────────────────┘
```

### 5. Session Continuity

Continue desktop sessions on mobile.

```
┌─────────────────────────────────────────┐
│ ← Active Sessions                  🔄   │
├─────────────────────────────────────────┤
│                                         │
│  Desktop Sessions                       │
│  ─────────────────────────────────────  │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ 💻 MacBook Pro                      ││
│  │ myproject • Active now              ││
│  │                                      ││
│  │ "Analyzing payment module..."       ││
│  │                                      ││
│  │ [Continue Here] [View Only]         ││
│  └─────────────────────────────────────┘│
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ 🖥️ Work Desktop                     ││
│  │ other-project • 2 hours ago         ││
│  │                                      ││
│  │ Last: "Generate API docs"           ││
│  │                                      ││
│  │ [Resume] [View History]             ││
│  └─────────────────────────────────────┘│
│                                         │
│  Mobile Sessions                        │
│  ─────────────────────────────────────  │
│                                         │
│  ┌─────────────────────────────────────┐│
│  │ 📱 This Device                      ││
│  │ myproject • 10 min ago              ││
│  │                                      ││
│  │ [New Session] [Continue]            ││
│  └─────────────────────────────────────┘│
│                                         │
└─────────────────────────────────────────┘
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Mobile App                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    React Native / Flutter                     │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │ Query    │ │ Notifi-  │ │ PR       │ │ Session  │        │   │
│  │  │ Screen   │ │ cations  │ │ Review   │ │ Manager  │        │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                    ┌─────────┴─────────┐                            │
│                    │ Amplifier SDK     │                            │
│                    │ (TypeScript)      │                            │
│                    └─────────┬─────────┘                            │
│                              │                                       │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                 │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐            │
│  │ API Server │      │ Push       │      │ Local      │            │
│  │            │      │ Service    │      │ Storage    │            │
│  └────────────┘      └────────────┘      └────────────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

```yaml
app:
  framework: React Native  # or Flutter
  language: TypeScript
  state: Zustand
  navigation: React Navigation

api:
  client: Generated from OpenAPI
  auth: OAuth2 + Biometric

notifications:
  ios: APNs
  android: FCM

offline:
  storage: AsyncStorage / SQLite
  sync: Background sync API

voice:
  ios: Speech Framework
  android: SpeechRecognizer
```

---

## Implementation

### App Structure (React Native)

```
src/
├── App.tsx
├── screens/
│   ├── HomeScreen.tsx
│   ├── QueryScreen.tsx
│   ├── QueryResultScreen.tsx
│   ├── NotificationsScreen.tsx
│   ├── PRReviewScreen.tsx
│   ├── SessionsScreen.tsx
│   └── SettingsScreen.tsx
├── components/
│   ├── QueryInput.tsx
│   ├── QueryResult.tsx
│   ├── NotificationCard.tsx
│   ├── PRSummary.tsx
│   ├── SessionCard.tsx
│   └── VoiceButton.tsx
├── services/
│   ├── amplifier.ts
│   ├── notifications.ts
│   ├── voice.ts
│   └── sync.ts
├── stores/
│   ├── queryStore.ts
│   ├── notificationStore.ts
│   └── sessionStore.ts
└── utils/
    ├── api.ts
    └── storage.ts
```

### Query Screen

```typescript
// screens/QueryScreen.tsx
import React, { useState } from 'react';
import {
  View,
  TextInput,
  FlatList,
  TouchableOpacity,
  Text,
  StyleSheet
} from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { VoiceButton } from '../components/VoiceButton';
import { useQueryStore } from '../stores/queryStore';
import { amplifier } from '../services/amplifier';

export function QueryScreen() {
  const [query, setQuery] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const navigation = useNavigation();
  const { recentQueries, addQuery } = useQueryStore();

  const handleSubmit = async () => {
    if (!query.trim()) return;

    setIsLoading(true);
    try {
      const result = await amplifier.execute({
        prompt: query,
        context: { type: 'mobile_query' }
      });

      addQuery({
        query,
        result: result.response,
        timestamp: new Date(),
        project: amplifier.currentProject
      });

      navigation.navigate('QueryResult', { result });
    } finally {
      setIsLoading(false);
    }
  };

  const handleVoiceResult = (transcript: string) => {
    setQuery(transcript);
    // Auto-submit voice queries
    handleSubmit();
  };

  return (
    <View style={styles.container}>
      <View style={styles.inputContainer}>
        <TextInput
          style={styles.input}
          value={query}
          onChangeText={setQuery}
          placeholder="Ask about your codebase..."
          multiline
          returnKeyType="send"
          onSubmitEditing={handleSubmit}
        />
        <VoiceButton onResult={handleVoiceResult} />
      </View>

      <Text style={styles.sectionTitle}>Recent Queries</Text>

      <FlatList
        data={recentQueries}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={styles.recentItem}
            onPress={() => navigation.navigate('QueryResult', { result: item })}
          >
            <Text style={styles.queryText}>{item.query}</Text>
            <Text style={styles.metaText}>
              {item.project} • {formatTimeAgo(item.timestamp)}
            </Text>
          </TouchableOpacity>
        )}
      />

      <TouchableOpacity
        style={[styles.askButton, isLoading && styles.askButtonDisabled]}
        onPress={handleSubmit}
        disabled={isLoading}
      >
        <Text style={styles.askButtonText}>
          {isLoading ? 'Asking...' : '🎤 Ask'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}
```

### Voice Input Component

```typescript
// components/VoiceButton.tsx
import React, { useState } from 'react';
import { TouchableOpacity, StyleSheet, Animated } from 'react-native';
import Voice from '@react-native-voice/voice';

interface VoiceButtonProps {
  onResult: (transcript: string) => void;
}

export function VoiceButton({ onResult }: VoiceButtonProps) {
  const [isListening, setIsListening] = useState(false);
  const pulseAnim = new Animated.Value(1);

  const startListening = async () => {
    setIsListening(true);

    // Start pulse animation
    Animated.loop(
      Animated.sequence([
        Animated.timing(pulseAnim, {
          toValue: 1.2,
          duration: 500,
          useNativeDriver: true
        }),
        Animated.timing(pulseAnim, {
          toValue: 1,
          duration: 500,
          useNativeDriver: true
        })
      ])
    ).start();

    Voice.onSpeechResults = (e) => {
      const transcript = e.value?.[0] || '';
      stopListening();
      onResult(transcript);
    };

    Voice.onSpeechError = () => {
      stopListening();
    };

    await Voice.start('en-US');
  };

  const stopListening = async () => {
    setIsListening(false);
    pulseAnim.stopAnimation();
    await Voice.stop();
  };

  return (
    <Animated.View style={{ transform: [{ scale: pulseAnim }] }}>
      <TouchableOpacity
        style={[
          styles.voiceButton,
          isListening && styles.voiceButtonActive
        ]}
        onPress={isListening ? stopListening : startListening}
      >
        <Text style={styles.voiceIcon}>🎤</Text>
      </TouchableOpacity>
    </Animated.View>
  );
}
```

### Notification Service

```typescript
// services/notifications.ts
import messaging from '@react-native-firebase/messaging';
import PushNotification from 'react-native-push-notification';
import { amplifier } from './amplifier';

export class NotificationService {
  async initialize() {
    // Request permission
    const authStatus = await messaging().requestPermission();

    if (authStatus === messaging.AuthorizationStatus.AUTHORIZED) {
      // Get FCM token
      const token = await messaging().getToken();
      await this.registerToken(token);

      // Handle token refresh
      messaging().onTokenRefresh(token => this.registerToken(token));

      // Handle foreground messages
      messaging().onMessage(async remoteMessage => {
        this.handleNotification(remoteMessage);
      });

      // Handle background messages
      messaging().setBackgroundMessageHandler(async remoteMessage => {
        this.handleNotification(remoteMessage);
      });
    }
  }

  async registerToken(token: string) {
    await amplifier.api.post('/v1/notifications/register', {
      token,
      platform: Platform.OS
    });
  }

  handleNotification(message: any) {
    const { type, data } = message.data;

    switch (type) {
      case 'build_failed':
        PushNotification.localNotification({
          title: '❌ Build Failed',
          message: `${data.project} • ${data.branch}`,
          data: { type, ...data }
        });
        break;

      case 'review_requested':
        PushNotification.localNotification({
          title: '🔔 Review Requested',
          message: `PR #${data.pr_number}: ${data.title}`,
          data: { type, ...data }
        });
        break;

      case 'deploy_complete':
        PushNotification.localNotification({
          title: '✅ Deploy Complete',
          message: `${data.project} • ${data.environment}`,
          data: { type, ...data }
        });
        break;
    }
  }
}
```

### PR Review Screen

```typescript
// screens/PRReviewScreen.tsx
import React, { useState, useEffect } from 'react';
import {
  View,
  ScrollView,
  Text,
  TouchableOpacity,
  TextInput,
  StyleSheet
} from 'react-native';
import { amplifier } from '../services/amplifier';

export function PRReviewScreen({ route }) {
  const { prNumber, repoUrl } = route.params;
  const [pr, setPR] = useState(null);
  const [aiSummary, setAISummary] = useState(null);
  const [comment, setComment] = useState('');
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    loadPR();
  }, []);

  const loadPR = async () => {
    setIsLoading(true);

    // Load PR details
    const prData = await amplifier.api.get(`/v1/github/pr/${prNumber}`);
    setPR(prData);

    // Get AI summary
    const summary = await amplifier.execute({
      prompt: `Summarize this PR for mobile review:\n\n${prData.diff}`,
      profile: 'enterprise-dev:mobile-review'
    });
    setAISummary(summary.response);

    setIsLoading(false);
  };

  const handleApprove = async () => {
    await amplifier.api.post(`/v1/github/pr/${prNumber}/approve`, {
      comment: comment || 'Approved via Amplifier Mobile'
    });
    navigation.goBack();
  };

  const handleComment = async () => {
    await amplifier.api.post(`/v1/github/pr/${prNumber}/comment`, {
      body: comment
    });
    setComment('');
  };

  const handleRequestChanges = async () => {
    await amplifier.api.post(`/v1/github/pr/${prNumber}/request-changes`, {
      comment
    });
    navigation.goBack();
  };

  if (isLoading) {
    return <LoadingScreen />;
  }

  return (
    <ScrollView style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>{pr.title}</Text>
        <Text style={styles.meta}>
          @{pr.author} → {pr.base} • +{pr.additions} -{pr.deletions}
        </Text>
      </View>

      <View style={styles.aiSummary}>
        <Text style={styles.sectionTitle}>🤖 Amplifier Summary</Text>
        <Text style={styles.summaryText}>{aiSummary}</Text>
      </View>

      <View style={styles.files}>
        <Text style={styles.sectionTitle}>Files ({pr.files.length})</Text>
        {pr.files.map(file => (
          <TouchableOpacity
            key={file.path}
            style={styles.fileItem}
            onPress={() => navigation.navigate('FileDiff', { file })}
          >
            <Text style={styles.fileName}>{file.path}</Text>
            <Text style={styles.fileChanges}>+{file.additions}</Text>
          </TouchableOpacity>
        ))}
      </View>

      <View style={styles.commentBox}>
        <TextInput
          style={styles.commentInput}
          value={comment}
          onChangeText={setComment}
          placeholder="Add a comment..."
          multiline
        />
      </View>

      <View style={styles.actions}>
        <TouchableOpacity
          style={[styles.actionButton, styles.approveButton]}
          onPress={handleApprove}
        >
          <Text style={styles.actionButtonText}>✅ Approve</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.actionButton, styles.commentButton]}
          onPress={handleComment}
        >
          <Text style={styles.actionButtonText}>💬 Comment</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.actionButton, styles.denyButton]}
          onPress={handleRequestChanges}
        >
          <Text style={styles.actionButtonText}>❌ Request Changes</Text>
        </TouchableOpacity>
      </View>
    </ScrollView>
  );
}
```

---

## Notification Types

| Type | Trigger | Actions |
|------|---------|---------|
| `build_failed` | CI build fails | View details, Ask Amplifier |
| `build_success` | CI build passes | View logs |
| `review_requested` | PR review assigned | Quick review, Open PR |
| `review_approved` | PR approved | View PR |
| `deploy_started` | Deployment begins | View progress |
| `deploy_complete` | Deployment finishes | View logs |
| `deploy_failed` | Deployment fails | View error, Ask Amplifier |
| `mention` | Mentioned in comment | View context, Reply |
| `session_update` | Desktop session update | Continue session |

---

## Offline Support

```typescript
// Offline-first architecture
const offlineConfig = {
  // Cache queries for offline viewing
  queryCaching: {
    enabled: true,
    maxItems: 100,
    maxAge: '7d'
  },

  // Queue actions when offline
  actionQueue: {
    enabled: true,
    syncOnReconnect: true
  },

  // Sync desktop sessions
  sessionSync: {
    enabled: true,
    backgroundSync: true,
    syncInterval: '5m'
  }
};
```

---

## Security

```yaml
security:
  auth:
    - oauth2 (GitHub, Google)
    - biometric (Face ID, fingerprint)
    - api_key

  data:
    - encrypted_storage (Keychain/Keystore)
    - no_code_cache (queries only)
    - auto_logout: 24h

  network:
    - certificate_pinning: true
    - min_tls: 1.3
```

---

## Events

| Event | Description | Data |
|-------|-------------|------|
| `mobile:query_sent` | Query submitted | query, voice |
| `mobile:notification_received` | Push received | type |
| `mobile:pr_reviewed` | PR action taken | action, pr |
| `mobile:session_continued` | Session resumed | session_id |

---

## Open Questions

1. **Offline queries**: Pre-cache common queries per project?
2. **Widget support**: iOS widgets for quick status?
3. **Watch app**: Apple Watch / Wear OS companion?
4. **Code viewing**: How to display code diffs effectively on mobile?

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | Draft | Initial specification |
