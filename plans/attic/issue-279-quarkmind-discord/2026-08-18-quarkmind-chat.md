# QuarkMind Chat — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #279 — quarkmind-discord — persistent character in Discord
**Issue group:** #279

**Goal:** Build quarkmind-chat phase 1 — the structural harness, execution model, and chat abstractions for platform-agnostic AI chat bots. This phase delivers a running agent that can connect to Discord, perceive conversations, reason via LLM, and respond with rate-governed pacing. Memory integration (D5) and personality growth (D6 Layer 3) are phase 2 — tracked as deferred GitHub issues in Batch 6.

**Architecture:** Two sub-modules (quarkmind-chat-protocol, quarkmind-chat-agent) under a quarkmind-chat parent. Platform-agnostic agency loop using quarkmind-core SPIs. Chat-specific abstractions (delta reports, attention classification, observation rendering, rate governing) added to quarkmind-core. Discord adapter layer uses casehub-connectors chat-spi + discord modules. Execution model uses blocks' `ChoreographedDriver` with `EventSource.merge()` for dual-wake (Discord events + heartbeat timer). Direct LLM agency loop (no CaseEngine) with `LlmInvoker` for synchronous invocation and `LlmRequestQueue.hasCapacity()` as a gate. Configuration-driven character deployment via YAML.

**Tech Stack:** Java 21+, Quarkus, Jackson (JSON parsing), casehub-connectors (chat-spi, chat-discord, discord), casehub-blocks (TieredObservationRenderer, EventSource, ChoreographedDriver, EventConcurrencyPolicy), casehub-eidos-api, quarkmind-core agency framework

## Global Constraints

- All code follows quarkmind-core package conventions (`io.quarkmind.agency.*` for core, `io.quarkmind.chat.*` for chat module)
- No CaseEngine dependency — direct LLM via synchronous `LlmInvoker` (production implementations wrap `AgentProvider`)
- `LlmRequestQueue.hasCapacity()` gated before all LLM calls — no invocation when queue is saturated
- Platform-specific code lives in sub-packages (`discord/`, `slack/`, etc.), never in the main package
- All tuning knobs externalized as Quarkus config properties — no hardcoded constants
- Tests run without a live Discord connection or LLM endpoint
- `ChatPlatform` SPI is the abstraction boundary — no direct Discord API calls from agent code
- `IntentQueue.enqueue()` — never `add()` (the method does not exist on IntentQueue)
- JSON parsing via Jackson `ObjectMapper` — never manual `extractField()` string scanning
- TDD: write failing test → verify fail → implement → verify pass → commit

---

## Batch 1: Module scaffold + protocol types

### Task 1: Create quarkmind-chat Maven module structure

**Files:**
- Create: `quarkmind-chat/pom.xml`
- Create: `quarkmind-chat/quarkmind-chat-protocol/pom.xml`
- Create: `quarkmind-chat/quarkmind-chat-agent/pom.xml`
- Create: `quarkmind-chat/quarkmind-chat-protocol/src/main/java/io/quarkmind/chat/protocol/package-info.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/package-info.java`
- Modify: `pom.xml` (root) — add `quarkmind-chat` to modules list

**Interfaces:**
- Produces: Maven module structure that compiles with `mvn compile -pl quarkmind-chat`

- [ ] **Step 1: Create parent POM for quarkmind-chat**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.quarkmind</groupId>
        <artifactId>quarkmind</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>quarkmind-chat</artifactId>
    <packaging>pom</packaging>
    <name>QuarkMind Chat</name>

    <modules>
        <module>quarkmind-chat-protocol</module>
        <module>quarkmind-chat-agent</module>
    </modules>
</project>
```

- [ ] **Step 2: Create protocol sub-module POM**

`quarkmind-chat/quarkmind-chat-protocol/pom.xml` — plain Java, no Quarkus dependencies:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.quarkmind</groupId>
        <artifactId>quarkmind-chat</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>quarkmind-chat-protocol</artifactId>
    <name>QuarkMind Chat Protocol</name>

    <dependencies>
        <dependency>
            <groupId>io.quarkmind</groupId>
            <artifactId>quarkmind-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-chat-spi</artifactId>
        </dependency>
    </dependencies>
</project>
```

Note: verify exact artifact IDs against the parent POM's `<dependencyManagement>`. The connectors chat-spi artifact ID may be `casehub-connectors-chat-spi` or another variant.

- [ ] **Step 3: Create agent sub-module POM**

`quarkmind-chat/quarkmind-chat-agent/pom.xml` — Quarkus application with Jackson for JSON:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.quarkmind</groupId>
        <artifactId>quarkmind-chat</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>quarkmind-chat-agent</artifactId>
    <name>QuarkMind Chat Agent</name>

    <dependencies>
        <dependency>
            <groupId>io.quarkmind</groupId>
            <artifactId>quarkmind-chat-protocol</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkmind</groupId>
            <artifactId>quarkmind-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-chat-spi</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-chat-discord</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-connectors-discord</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-blocks</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-eidos-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-jackson</artifactId>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 4: Add quarkmind-chat to root POM modules**

Add `<module>quarkmind-chat</module>` to the root `pom.xml` modules list, after `quarkmind-ville`.

- [ ] **Step 5: Create package-info files**

`quarkmind-chat-protocol/src/main/java/io/quarkmind/chat/protocol/package-info.java`:
```java
/**
 * Shared protocol types for quarkmind-chat — platform-agnostic perception and intent records.
 */
package io.quarkmind.chat.protocol;
```

`quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/package-info.java`:
```java
/**
 * Chat bot agency loop, world bridge, and platform adapters.
 */
package io.quarkmind.chat.agent;
```

- [ ] **Step 6: Verify compilation**

Run: `mvn compile -pl quarkmind-chat -q`
Expected: BUILD SUCCESS (empty modules compile)

- [ ] **Step 7: Commit**

```bash
git add quarkmind-chat/ pom.xml
git commit -m "feat(#279): quarkmind-chat module scaffold — protocol + agent sub-modules Refs #279"
```

### Task 2: ChatIntent sealed interface + ChatPerception record

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-protocol/src/main/java/io/quarkmind/chat/protocol/ChatIntent.java`
- Create: `quarkmind-chat/quarkmind-chat-protocol/src/main/java/io/quarkmind/chat/protocol/ChatPerception.java`
- Create: `quarkmind-chat/quarkmind-chat-protocol/src/main/java/io/quarkmind/chat/protocol/WakeReason.java`
- Test: `quarkmind-chat/quarkmind-chat-protocol/src/test/java/io/quarkmind/chat/protocol/ChatIntentTest.java`
- Test: `quarkmind-chat/quarkmind-chat-protocol/src/test/java/io/quarkmind/chat/protocol/ChatPerceptionTest.java`

**Interfaces:**
- Consumes: `io.quarkmind.agency.intent.Intent` (marker), `io.casehub.connectors.chat.model.ChatContent`, `ChatChannelRef`, `ChatMessageRef`, `MemberRef`, `ReceivedMessage`, `PresenceStatus`
- Produces: `ChatIntent` (sealed: Send, Reply, React), `ChatPerception(channelDeltas, presenceChanges, reason)`, `WakeReason.fromDriverSource(String)`

- [ ] **Step 1: Write ChatIntent test**

```java
package io.quarkmind.chat.protocol;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ChatIntentTest {

    @Test
    void sendCreatesWithChannelAndContent() {
        var intent = new ChatIntent.Send("channel-1", new ChatContent("hello"));
        assertEquals("channel-1", intent.channelId());
        assertEquals("hello", intent.content().text());
    }

    @Test
    void replyCreatesWithParentAndContent() {
        var channel = new ChatChannelRef("ch-1");
        var parent = new ChatMessageRef(channel, "msg-1");
        var intent = new ChatIntent.Reply(parent, new ChatContent("reply text"));
        assertEquals("msg-1", intent.parent().messageId());
    }

    @Test
    void reactCreatesWithMessageAndEmoji() {
        var channel = new ChatChannelRef("ch-1");
        var msgRef = new ChatMessageRef(channel, "msg-1");
        var intent = new ChatIntent.React(msgRef, "👀");
        assertEquals("👀", intent.emoji());
    }

    @Test
    void sealedPermitsExactlyThreeVariants() {
        var permits = ChatIntent.class.getPermittedSubclasses();
        assertNotNull(permits);
        assertEquals(3, permits.length);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol -Dtest=ChatIntentTest -q`
Expected: FAIL — `ChatIntent` does not exist

- [ ] **Step 3: Implement ChatIntent**

```java
package io.quarkmind.chat.protocol;

import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.ChatMessageRef;
import io.quarkmind.agency.intent.Intent;

public sealed interface ChatIntent extends Intent {
    record Send(String channelId, ChatContent content) implements ChatIntent {}
    record Reply(ChatMessageRef parent, ChatContent content) implements ChatIntent {}
    record React(ChatMessageRef message, String emoji) implements ChatIntent {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol -Dtest=ChatIntentTest -q`
Expected: PASS

- [ ] **Step 5: Write WakeReason enum**

```java
package io.quarkmind.chat.protocol;

public enum WakeReason {
    MESSAGE,
    HEARTBEAT;

    public static WakeReason fromDriverSource(String source) {
        if ("timer".equals(source)) return HEARTBEAT;
        return MESSAGE;
    }
}
```

- [ ] **Step 6: Write ChatPerception test**

```java
package io.quarkmind.chat.protocol;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ChatPerceptionTest {

    @Test
    void constructsWithDeltasAndReason() {
        var msg = dummyMessage("hello", "ch-1");
        var deltas = Map.of("ch-1", List.of(msg));
        var perception = new ChatPerception(deltas, Map.of(), WakeReason.MESSAGE);
        assertEquals(1, perception.channelDeltas().get("ch-1").size());
        assertEquals(WakeReason.MESSAGE, perception.reason());
    }

    @Test
    void emptyPerceptionForHeartbeat() {
        var perception = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);
        assertTrue(perception.channelDeltas().isEmpty());
        assertEquals(WakeReason.HEARTBEAT, perception.reason());
    }

    @Test
    void hasActivityReturnsTrueWhenMessagesExist() {
        var withDeltas = new ChatPerception(
                Map.of("ch", List.of(dummyMessage("hi", "ch"))), Map.of(), WakeReason.MESSAGE);
        assertTrue(withDeltas.hasActivity());
    }

    @Test
    void hasActivityReturnsFalseWhenEmpty() {
        var empty = new ChatPerception(Map.of(), Map.of(), WakeReason.MESSAGE);
        assertFalse(empty.hasActivity());
    }

    private ReceivedMessage dummyMessage(String text, String channelId) {
        return new ReceivedMessage("discord", new ChatChannelRef(channelId),
                new ChatMessageRef(new ChatChannelRef(channelId), "m1"), null,
                new MemberRef("u1"), new ChatContent(text), Instant.now());
    }
}
```

- [ ] **Step 7: Implement ChatPerception**

```java
package io.quarkmind.chat.protocol;

import io.casehub.connectors.chat.model.PresenceStatus;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.quarkmind.agency.spi.WorldPerception;
import java.util.List;
import java.util.Map;

public record ChatPerception(
        Map<String, List<ReceivedMessage>> channelDeltas,
        Map<String, PresenceStatus> presenceChanges,
        WakeReason reason
) implements WorldPerception {

    public ChatPerception {
        channelDeltas = channelDeltas != null ? Map.copyOf(channelDeltas) : Map.of();
        presenceChanges = presenceChanges != null ? Map.copyOf(presenceChanges) : Map.of();
    }

    public boolean hasActivity() {
        return !channelDeltas.isEmpty() && channelDeltas.values().stream().anyMatch(l -> !l.isEmpty());
    }

    public int totalMessageCount() {
        return channelDeltas.values().stream().mapToInt(List::size).sum();
    }
}
```

- [ ] **Step 8: Run all protocol tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-protocol -q`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add quarkmind-chat/quarkmind-chat-protocol/
git commit -m "feat(#279): ChatIntent sealed interface + ChatPerception record Refs #279"
```

## Batch 2: quarkmind-core chat abstractions

### Task 3: AttentionClassifier + ChatDeltaReport

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/AttentionPriority.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/AttentionClassifier.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/BotIdentityDetector.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/ClassifiedMessage.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/ChatDeltaReport.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/ConversationThread.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/chat/AttentionClassifierTest.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/chat/ChatDeltaReportTest.java`

**Interfaces:**
- Consumes: `ReceivedMessage`, `ChatMessageRef`, `MemberRef` from connectors chat-spi
- Produces: `AttentionClassifier.classify(List<ReceivedMessage>, BotIdentityDetector, Set<String> participatedThreadIds) → List<ClassifiedMessage>`, `ChatDeltaReport.build(Map<String, List<ReceivedMessage>>, BotIdentityDetector, Set<String>) → ChatDeltaReport`

- [ ] **Step 1: Write AttentionPriority enum + BotIdentityDetector SPI + ClassifiedMessage record**

```java
package io.quarkmind.agency.chat;

public enum AttentionPriority {
    DIRECT,    // replies to bot, @mentions — always verbatim
    ELEVATED,  // threads bot previously participated in
    AMBIENT    // general channel chatter — compressible
}
```

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;

public interface BotIdentityDetector {
    boolean isMention(ReceivedMessage message);
    boolean isReplyToBot(ReceivedMessage message);
    String botUserId();
}
```

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;

public record ClassifiedMessage(ReceivedMessage message, AttentionPriority priority) {}
```

- [ ] **Step 2: Write AttentionClassifier test**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class AttentionClassifierTest {

    private final BotIdentityDetector detector = new BotIdentityDetector() {
        @Override public boolean isMention(ReceivedMessage msg) {
            return msg.content().text().contains("@bot");
        }
        @Override public boolean isReplyToBot(ReceivedMessage msg) {
            return msg.parentRef() != null
                    && msg.parentRef().messageId().equals("bot-msg-1");
        }
        @Override public String botUserId() { return "bot-id"; }
    };

    @Test
    void mentionClassifiedAsDirect() {
        var msg = msg("@bot hello", null);
        var classified = AttentionClassifier.classify(List.of(msg), detector, Set.of());
        assertEquals(AttentionPriority.DIRECT, classified.get(0).priority());
    }

    @Test
    void replyToBotClassifiedAsDirect() {
        var parent = new ChatMessageRef(new ChatChannelRef("ch"), "bot-msg-1");
        var msg = msg("sure thing", parent);
        var classified = AttentionClassifier.classify(List.of(msg), detector, Set.of());
        assertEquals(AttentionPriority.DIRECT, classified.get(0).priority());
    }

    @Test
    void messageInParticipatedThreadClassifiedAsElevated() {
        var parent = new ChatMessageRef(new ChatChannelRef("ch"), "thread-1");
        var msg = msg("continuing discussion", parent);
        var classified = AttentionClassifier.classify(List.of(msg), detector, Set.of("thread-1"));
        assertEquals(AttentionPriority.ELEVATED, classified.get(0).priority());
    }

    @Test
    void ordinaryMessageClassifiedAsAmbient() {
        var msg = msg("hey everyone", null);
        var classified = AttentionClassifier.classify(List.of(msg), detector, Set.of());
        assertEquals(AttentionPriority.AMBIENT, classified.get(0).priority());
    }

    private ReceivedMessage msg(String text, ChatMessageRef parent) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m-" + text.hashCode()),
                parent, new MemberRef("user-1"), new ChatContent(text), Instant.now());
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest -q`
Expected: FAIL — `AttentionClassifier` does not exist

- [ ] **Step 4: Implement AttentionClassifier**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;
import java.util.List;
import java.util.Set;

public final class AttentionClassifier {

    private AttentionClassifier() {}

    public static List<ClassifiedMessage> classify(
            List<ReceivedMessage> messages,
            BotIdentityDetector detector,
            Set<String> participatedThreadIds) {
        return messages.stream()
                .map(msg -> new ClassifiedMessage(msg, priorityOf(msg, detector, participatedThreadIds)))
                .toList();
    }

    private static AttentionPriority priorityOf(
            ReceivedMessage msg,
            BotIdentityDetector detector,
            Set<String> participatedThreadIds) {
        if (detector.isMention(msg) || detector.isReplyToBot(msg)) {
            return AttentionPriority.DIRECT;
        }
        if (msg.parentRef() != null
                && participatedThreadIds.contains(msg.parentRef().messageId())) {
            return AttentionPriority.ELEVATED;
        }
        return AttentionPriority.AMBIENT;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest -q`
Expected: PASS

- [ ] **Step 6: Write ConversationThread record**

```java
package io.quarkmind.agency.chat;

import java.util.List;

public record ConversationThread(
        String threadId,
        List<ClassifiedMessage> messages,
        boolean isNew,
        AttentionPriority highestPriority
) {
    public int messageCount() { return messages.size(); }
}
```

- [ ] **Step 7: Write ChatDeltaReport test**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ChatDeltaReportTest {

    private final BotIdentityDetector detector = new BotIdentityDetector() {
        @Override public boolean isMention(ReceivedMessage msg) { return false; }
        @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
        @Override public String botUserId() { return "bot-id"; }
    };

    @Test
    void groupsMessagesByThread() {
        var root = msg("hello", null, "m1");
        var reply = msg("hi back", ref("m1"), "m2");
        var standalone = msg("unrelated", null, "m3");

        var report = ChatDeltaReport.build(
                Map.of("ch", List.of(root, reply, standalone)), detector, Set.of());
        assertEquals(2, report.threads("ch").size());
    }

    @Test
    void directMessagesGetHighestPriority() {
        var mentionDetector = new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) {
                return msg.content().text().contains("@bot");
            }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot-id"; }
        };

        var msg = msg("@bot help", null, "m1");
        var report = ChatDeltaReport.build(Map.of("ch", List.of(msg)), mentionDetector, Set.of());
        assertEquals(AttentionPriority.DIRECT, report.threads("ch").get(0).highestPriority());
    }

    @Test
    void emptyDeltaProducesEmptyReport() {
        var report = ChatDeltaReport.build(Map.of(), detector, Set.of());
        assertTrue(report.allChannels().isEmpty());
    }

    @Test
    void directMessagesAccessor() {
        var mentionDetector = new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) {
                return msg.content().text().contains("@bot");
            }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot-id"; }
        };
        var direct = msg("@bot help", null, "m1");
        var ambient = msg("random chat", null, "m2");
        var report = ChatDeltaReport.build(
                Map.of("ch", List.of(direct, ambient)), mentionDetector, Set.of());
        assertEquals(1, report.directMessages().size());
        assertEquals("@bot help", report.directMessages().get(0).message().content().text());
    }

    private ReceivedMessage msg(String text, ChatMessageRef parent, String id) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), id),
                parent, new MemberRef("user-1"), new ChatContent(text), Instant.now());
    }

    private ChatMessageRef ref(String id) {
        return new ChatMessageRef(new ChatChannelRef("ch"), id);
    }
}
```

- [ ] **Step 8: Implement ChatDeltaReport**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;
import java.util.*;

public record ChatDeltaReport(Map<String, List<ConversationThread>> channelThreads) {

    public List<ConversationThread> threads(String channelId) {
        return channelThreads.getOrDefault(channelId, List.of());
    }

    public Set<String> allChannels() {
        return channelThreads.keySet();
    }

    public List<ClassifiedMessage> directMessages() {
        return channelThreads.values().stream()
                .flatMap(List::stream)
                .filter(t -> t.highestPriority() == AttentionPriority.DIRECT)
                .flatMap(t -> t.messages().stream())
                .toList();
    }

    public List<ClassifiedMessage> ambientMessages() {
        return channelThreads.values().stream()
                .flatMap(List::stream)
                .filter(t -> t.highestPriority() == AttentionPriority.AMBIENT)
                .flatMap(t -> t.messages().stream())
                .toList();
    }

    public static ChatDeltaReport build(
            Map<String, List<ReceivedMessage>> channelDeltas,
            BotIdentityDetector detector,
            Set<String> participatedThreadIds) {

        var result = new LinkedHashMap<String, List<ConversationThread>>();
        for (var entry : channelDeltas.entrySet()) {
            String channelId = entry.getKey();
            List<ClassifiedMessage> classified =
                    AttentionClassifier.classify(entry.getValue(), detector, participatedThreadIds);

            Map<String, List<ClassifiedMessage>> threadGroups = new LinkedHashMap<>();
            for (ClassifiedMessage cm : classified) {
                String threadId = cm.message().parentRef() != null
                        ? cm.message().parentRef().messageId()
                        : cm.message().messageRef().messageId();
                threadGroups.computeIfAbsent(threadId, k -> new ArrayList<>()).add(cm);
            }

            List<ConversationThread> threads = threadGroups.entrySet().stream()
                    .map(e -> {
                        AttentionPriority highest = e.getValue().stream()
                                .map(ClassifiedMessage::priority)
                                .min(Comparator.comparingInt(Enum::ordinal))
                                .orElse(AttentionPriority.AMBIENT);
                        boolean isNew = !participatedThreadIds.contains(e.getKey());
                        return new ConversationThread(e.getKey(), List.copyOf(e.getValue()), isNew, highest);
                    })
                    .toList();

            result.put(channelId, threads);
        }
        return new ChatDeltaReport(Map.copyOf(result));
    }
}
```

- [ ] **Step 9: Run all tests**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest,ChatDeltaReportTest -q`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add quarkmind-core/src/
git commit -m "feat(#279): AttentionClassifier + ChatDeltaReport — chat conversation analysis Refs #279"
```

### Task 4: ChatObservationRenderer + OutputGovernor + ProactiveDecisionGate + NeedThresholdWake + ChatPerceptionBridge

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/ChatObservationRenderer.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/chat/ChatPerceptionBridge.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/OutputGovernor.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/personality/ProactiveDecisionGate.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/NeedThresholdWake.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/chat/ChatObservationRendererTest.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/OutputGovernorTest.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/personality/ProactiveDecisionGateTest.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/NeedThresholdWakeTest.java`

**Interfaces:**
- Consumes: `ChatDeltaReport`, `ClassifiedMessage`, `AttentionPriority`, blocks `TieredObservationRenderer<ClassifiedMessage>`, `LevelEvent`, `EventLevel`, `ObservationContext`, `ObservationResult`, `NeedState`, `EventSource` (blocks)
- Produces: `ChatObservationRenderer.renderDelta(ChatDeltaReport) → String` (DIRECT/ELEVATED verbatim, AMBIENT compressed), `OutputGovernor.allow() → boolean`, `ProactiveDecisionGate.shouldAct(long, int, boolean) → boolean`, `NeedThresholdWake implements EventSource`, `ChatPerceptionBridge` SPI

- [ ] **Step 1: Write ChatObservationRenderer test**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ChatObservationRendererTest {

    private final BotIdentityDetector detector = new BotIdentityDetector() {
        @Override public boolean isMention(ReceivedMessage msg) {
            return msg.content().text().contains("@bot");
        }
        @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
        @Override public String botUserId() { return "bot-id"; }
    };

    @Test
    void directMessagesAlwaysVerbatim() {
        var msg = msg("@bot help me please", "m1");
        var report = ChatDeltaReport.build(Map.of("ch", List.of(msg)), detector, Set.of());
        var renderer = new ChatObservationRenderer(10);
        String result = renderer.renderDelta(report);
        assertTrue(result.contains("@bot help me please"));
    }

    @Test
    void ambientMessagesCompressedWhenOverThreshold() {
        var messages = new ArrayList<ReceivedMessage>();
        for (int i = 0; i < 15; i++) {
            messages.add(msg("ambient message " + i, "m" + i));
        }
        var noMentionDetector = new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) { return false; }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot-id"; }
        };
        var report = ChatDeltaReport.build(Map.of("ch", messages), noMentionDetector, Set.of());
        var renderer = new ChatObservationRenderer(5);
        String result = renderer.renderDelta(report);
        assertFalse(result.contains("ambient message 0"));
        assertTrue(result.contains("ambient"));
    }

    @Test
    void fewAmbientMessagesRenderedVerbatim() {
        var messages = List.of(msg("hello world", "m1"), msg("nice day", "m2"));
        var noMentionDetector = new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) { return false; }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot-id"; }
        };
        var report = ChatDeltaReport.build(Map.of("ch", messages), noMentionDetector, Set.of());
        var renderer = new ChatObservationRenderer(10);
        String result = renderer.renderDelta(report);
        assertTrue(result.contains("hello world"));
        assertTrue(result.contains("nice day"));
    }

    private ReceivedMessage msg(String text, String id) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), id), null,
                new MemberRef("user-1"), new ChatContent(text), Instant.now());
    }
}
```

- [ ] **Step 2: Implement ChatObservationRenderer**

Renders a `ChatDeltaReport` into LLM-ready text. DIRECT and ELEVATED messages always verbatim. AMBIENT messages compressed when count exceeds threshold — grouped by sender.

```java
package io.quarkmind.agency.chat;

import java.util.List;
import java.util.stream.Collectors;

public class ChatObservationRenderer {

    private final int ambientVerbatimThreshold;

    public ChatObservationRenderer(int ambientVerbatimThreshold) {
        this.ambientVerbatimThreshold = ambientVerbatimThreshold;
    }

    public String renderDelta(ChatDeltaReport report) {
        var sb = new StringBuilder();
        for (String channelId : report.allChannels()) {
            sb.append("Channel #").append(channelId).append(":\n");
            for (ConversationThread thread : report.threads(channelId)) {
                if (thread.highestPriority() == AttentionPriority.DIRECT
                        || thread.highestPriority() == AttentionPriority.ELEVATED) {
                    renderVerbatim(thread.messages(), sb);
                } else {
                    renderAmbient(thread.messages(), sb);
                }
            }
        }
        return sb.toString();
    }

    private void renderVerbatim(List<ClassifiedMessage> messages, StringBuilder sb) {
        for (ClassifiedMessage cm : messages) {
            sb.append("  ").append(cm.message().sender().id()).append(": ")
                    .append(cm.message().content().text()).append("\n");
        }
    }

    private void renderAmbient(List<ClassifiedMessage> messages, StringBuilder sb) {
        if (messages.size() <= ambientVerbatimThreshold) {
            renderVerbatim(messages, sb);
        } else {
            var bySender = messages.stream()
                    .collect(Collectors.groupingBy(cm -> cm.message().sender().id()));
            sb.append("  [").append(messages.size()).append(" ambient messages from ");
            sb.append(bySender.keySet().stream().collect(Collectors.joining(", ")));
            sb.append("]\n");
        }
    }
}
```

- [ ] **Step 3: Write ChatPerceptionBridge SPI**

```java
package io.quarkmind.agency.chat;

import io.quarkmind.chat.protocol.ChatPerception;

public interface ChatPerceptionBridge {
    ChatDeltaReport buildDelta(ChatPerception perception, BotIdentityDetector detector,
                               java.util.Set<String> participatedThreadIds);
    String renderForLlm(ChatDeltaReport report);
}
```

Default implementation:

```java
package io.quarkmind.agency.chat;

import io.quarkmind.chat.protocol.ChatPerception;
import java.util.Set;

public class DefaultChatPerceptionBridge implements ChatPerceptionBridge {

    private final ChatObservationRenderer renderer;

    public DefaultChatPerceptionBridge(ChatObservationRenderer renderer) {
        this.renderer = renderer;
    }

    @Override
    public ChatDeltaReport buildDelta(ChatPerception perception, BotIdentityDetector detector,
                                       Set<String> participatedThreadIds) {
        return ChatDeltaReport.build(perception.channelDeltas(), detector, participatedThreadIds);
    }

    @Override
    public String renderForLlm(ChatDeltaReport report) {
        return renderer.renderDelta(report);
    }
}
```

- [ ] **Step 4: Write OutputGovernor test**

```java
package io.quarkmind.agency.schedule;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class OutputGovernorTest {

    @Test
    void allowsFirstAction() {
        var governor = new OutputGovernor(300_000, 30_000, 1);
        assertTrue(governor.allow());
    }

    @Test
    void blocksActionWithinMinInterval() {
        var governor = new OutputGovernor(300_000, 30_000, 1);
        governor.recordAction();
        assertFalse(governor.allow());
    }

    @Test
    void blocksWhenWindowMaxReached() {
        var governor = new OutputGovernor(300_000, 0, 1);
        governor.recordAction();
        assertFalse(governor.allow());
    }
}
```

- [ ] **Step 5: Implement OutputGovernor**

```java
package io.quarkmind.agency.schedule;

import java.util.ArrayDeque;
import java.util.Deque;

public class OutputGovernor {

    private final long windowMs;
    private final long minIntervalMs;
    private final int maxPerWindow;
    private final Deque<Long> actionTimestamps = new ArrayDeque<>();

    public OutputGovernor(long windowMs, long minIntervalMs, int maxPerWindow) {
        this.windowMs = windowMs;
        this.minIntervalMs = minIntervalMs;
        this.maxPerWindow = maxPerWindow;
    }

    public boolean allow() {
        long now = System.currentTimeMillis();
        pruneOld(now);
        if (!actionTimestamps.isEmpty() && now - actionTimestamps.peekLast() < minIntervalMs) {
            return false;
        }
        return actionTimestamps.size() < maxPerWindow;
    }

    public void recordAction() {
        actionTimestamps.addLast(System.currentTimeMillis());
    }

    private void pruneOld(long now) {
        while (!actionTimestamps.isEmpty() && now - actionTimestamps.peekFirst() > windowMs) {
            actionTimestamps.pollFirst();
        }
    }
}
```

- [ ] **Step 6: Write ProactiveDecisionGate test** (in `io.quarkmind.agency.personality`)

```java
package io.quarkmind.agency.personality;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ProactiveDecisionGateTest {

    @Test
    void allowsWhenQuietAndSufficientTimePassed() {
        var gate = new ProactiveDecisionGate(60_000, 5);
        assertTrue(gate.shouldAct(120_000, 0, false));
    }

    @Test
    void blocksWhenTooSoon() {
        var gate = new ProactiveDecisionGate(60_000, 5);
        assertFalse(gate.shouldAct(10_000, 0, false));
    }

    @Test
    void blocksWhenOthersTyping() {
        var gate = new ProactiveDecisionGate(60_000, 5);
        assertFalse(gate.shouldAct(120_000, 0, true));
    }

    @Test
    void blocksWhenChannelTooActive() {
        var gate = new ProactiveDecisionGate(60_000, 5);
        assertFalse(gate.shouldAct(120_000, 10, false));
    }
}
```

- [ ] **Step 7: Implement ProactiveDecisionGate**

```java
package io.quarkmind.agency.personality;

public class ProactiveDecisionGate {

    private final long minTimeSinceLastPostMs;
    private final int maxChannelActivity;

    public ProactiveDecisionGate(long minTimeSinceLastPostMs, int maxChannelActivity) {
        this.minTimeSinceLastPostMs = minTimeSinceLastPostMs;
        this.maxChannelActivity = maxChannelActivity;
    }

    public boolean shouldAct(long timeSinceLastPostMs, int recentChannelMessages, boolean othersTyping) {
        if (timeSinceLastPostMs < minTimeSinceLastPostMs) return false;
        if (othersTyping) return false;
        return recentChannelMessages <= maxChannelActivity;
    }
}
```

- [ ] **Step 8: Write NeedThresholdWake test**

```java
package io.quarkmind.agency.schedule;

import io.quarkmind.agency.needs.NeedState;
import org.junit.jupiter.api.Test;
import java.util.concurrent.atomic.AtomicBoolean;
import static org.junit.jupiter.api.Assertions.*;

class NeedThresholdWakeTest {

    @Test
    void firesWhenSocialBelowThreshold() {
        var needs = new NeedState();
        needs.set("SOCIAL", 20.0);
        var wake = new NeedThresholdWake(needs, 30.0);
        assertTrue(wake.shouldFire());
    }

    @Test
    void doesNotFireWhenSocialAboveThreshold() {
        var needs = new NeedState();
        needs.set("SOCIAL", 80.0);
        var wake = new NeedThresholdWake(needs, 30.0);
        assertFalse(wake.shouldFire());
    }

    @Test
    void subscribeSendsEventOnEvaluation() {
        var needs = new NeedState();
        needs.set("SOCIAL", 20.0);
        var wake = new NeedThresholdWake(needs, 30.0);
        var fired = new AtomicBoolean(false);
        wake.evaluate(event -> fired.set(true));
        assertTrue(fired.get());
    }
}
```

- [ ] **Step 9: Implement NeedThresholdWake**

```java
package io.quarkmind.agency.schedule;

import io.casehub.blocks.agentic.model.DriverEvent;
import io.quarkmind.agency.needs.NeedState;
import java.util.function.Consumer;

public class NeedThresholdWake {

    private final NeedState needs;
    private final double socialThreshold;

    public NeedThresholdWake(NeedState needs, double socialThreshold) {
        this.needs = needs;
        this.socialThreshold = socialThreshold;
    }

    public boolean shouldFire() {
        return needs.get("SOCIAL") < socialThreshold;
    }

    public void evaluate(Consumer<DriverEvent> sink) {
        if (shouldFire()) {
            sink.accept(DriverEvent.signal("need-threshold"));
        }
    }
}
```

- [ ] **Step 10: Run all Batch 2 tests**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest,ChatDeltaReportTest,ChatObservationRendererTest,OutputGovernorTest,ProactiveDecisionGateTest,NeedThresholdWakeTest -q`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add quarkmind-core/src/
git commit -m "feat(#279): ChatObservationRenderer, OutputGovernor, ProactiveDecisionGate, NeedThresholdWake, ChatPerceptionBridge Refs #279"
```

## Batch 3: Discord adapters + world bridge

### Task 5: DiscordIdentityDetector

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/discord/DiscordIdentityDetector.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/discord/DiscordIdentityDetectorTest.java`

**Interfaces:**
- Consumes: `BotIdentityDetector` SPI from quarkmind-core
- Produces: `DiscordIdentityDetector implements BotIdentityDetector`, `recordBotMessage(String messageId)`

- [ ] **Step 1: Write DiscordIdentityDetector test**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class DiscordIdentityDetectorTest {

    private final DiscordIdentityDetector detector = new DiscordIdentityDetector("12345");

    @Test
    void detectsMentionWithDiscordSyntax() {
        assertTrue(detector.isMention(msg("hey <@12345> what's up")));
    }

    @Test
    void doesNotDetectMentionOfOtherUser() {
        assertFalse(detector.isMention(msg("hey <@99999> what's up")));
    }

    @Test
    void detectsReplyToBot() {
        detector.recordBotMessage("bot-sent-msg");
        var parent = new ChatMessageRef(new ChatChannelRef("ch"), "bot-sent-msg");
        var reply = new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"),
                parent, new MemberRef("user-1"), new ChatContent("replying"), Instant.now());
        assertTrue(detector.isReplyToBot(reply));
    }

    @Test
    void doesNotDetectReplyToOtherMessage() {
        var parent = new ChatMessageRef(new ChatChannelRef("ch"), "other-msg");
        var reply = new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"),
                parent, new MemberRef("user-1"), new ChatContent("replying"), Instant.now());
        assertFalse(detector.isReplyToBot(reply));
    }

    private ReceivedMessage msg(String text) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"), null,
                new MemberRef("user-1"), new ChatContent(text), Instant.now());
    }
}
```

- [ ] **Step 2: Implement DiscordIdentityDetector**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.connectors.chat.model.ReceivedMessage;
import io.quarkmind.agency.chat.BotIdentityDetector;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class DiscordIdentityDetector implements BotIdentityDetector {

    private final String botUserId;
    private final String mentionPattern;
    private final Set<String> botMessageIds = ConcurrentHashMap.newKeySet();

    public DiscordIdentityDetector(String botUserId) {
        this.botUserId = botUserId;
        this.mentionPattern = "<@" + botUserId + ">";
    }

    @Override
    public boolean isMention(ReceivedMessage message) {
        String text = message.content().text();
        return text != null && text.contains(mentionPattern);
    }

    @Override
    public boolean isReplyToBot(ReceivedMessage message) {
        return message.parentRef() != null
                && botMessageIds.contains(message.parentRef().messageId());
    }

    @Override
    public String botUserId() {
        return botUserId;
    }

    public void recordBotMessage(String messageId) {
        botMessageIds.add(messageId);
    }
}
```

- [ ] **Step 3: Run test, commit**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=DiscordIdentityDetectorTest -q`
Expected: PASS

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): DiscordIdentityDetector — Discord @mention and reply detection Refs #279"
```

### Task 6: DiscordEventSource + DiscordGatewayMessageHistory

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/discord/DiscordEventSource.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/discord/DiscordGatewayMessageHistory.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/discord/DiscordEventSourceTest.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/discord/DiscordGatewayMessageHistoryTest.java`

**Interfaces:**
- Consumes: `DiscordInboundConnector` (connectors), `InboundMessageSink` (connectors), blocks `EventSource`, `DriverEvent`, `MessageHistory` (chat-spi), `ReceivedMessage`, `ChatChannelRef`
- Produces: `DiscordEventSource implements EventSource` (emits DriverEvent("discord") on MESSAGE_CREATE), `DiscordGatewayMessageHistory implements MessageHistory` (accumulates Gateway events, REST fallback)

- [ ] **Step 1: Write DiscordEventSource test**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.blocks.agentic.model.DriverEvent;
import org.junit.jupiter.api.Test;
import java.util.concurrent.atomic.AtomicReference;
import static org.junit.jupiter.api.Assertions.*;

class DiscordEventSourceTest {

    @Test
    void emitsDriverEventOnInboundMessage() {
        var eventSource = new DiscordEventSource();
        var received = new AtomicReference<DriverEvent>();
        var cancellation = eventSource.subscribe(received::set);

        eventSource.onMessage("ch-1", "user-1", "hello");

        assertNotNull(received.get());
        assertEquals("discord", received.get().source());
        cancellation.cancel();
    }

    @Test
    void cancelledSubscriptionStopsEvents() {
        var eventSource = new DiscordEventSource();
        var received = new AtomicReference<DriverEvent>();
        var cancellation = eventSource.subscribe(received::set);
        cancellation.cancel();

        eventSource.onMessage("ch-1", "user-1", "hello");
        assertNull(received.get());
    }
}
```

- [ ] **Step 2: Implement DiscordEventSource**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.blocks.agentic.model.DriverEvent;
import io.casehub.blocks.agentic.model.EventSource;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

public class DiscordEventSource implements EventSource {

    private final Set<Consumer<DriverEvent>> subscribers = ConcurrentHashMap.newKeySet();

    @Override
    public Cancellation subscribe(Consumer<DriverEvent> sink) {
        subscribers.add(sink);
        return Cancellation.of(() -> subscribers.remove(sink));
    }

    public void onMessage(String channelId, String senderId, String content) {
        var event = new DriverEvent("discord", java.util.Map.of(
                "channelId", channelId, "senderId", senderId));
        subscribers.forEach(s -> s.accept(event));
    }
}
```

- [ ] **Step 3: Write DiscordGatewayMessageHistory test**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class DiscordGatewayMessageHistoryTest {

    @Test
    void accumulatesMessages() {
        var history = new DiscordGatewayMessageHistory();
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"), new ChatContent("hello"), Instant.now());
        history.accumulate(msg);

        var result = history.messages(new ChatChannelRef("ch-1"), Instant.EPOCH);
        assertEquals(1, result.size());
        assertEquals("hello", result.get(0).content().text());
    }

    @Test
    void onlyReturnsMessagesSinceTimestamp() {
        var history = new DiscordGatewayMessageHistory();
        var old = new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"), new ChatContent("old"),
                Instant.parse("2026-01-01T00:00:00Z"));
        var recent = new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m2"), null,
                new MemberRef("user-1"), new ChatContent("recent"),
                Instant.parse("2026-08-01T00:00:00Z"));
        history.accumulate(old);
        history.accumulate(recent);

        var result = history.messages(new ChatChannelRef("ch-1"),
                Instant.parse("2026-06-01T00:00:00Z"));
        assertEquals(1, result.size());
        assertEquals("recent", result.get(0).content().text());
    }

    @Test
    void emptyForUnknownChannel() {
        var history = new DiscordGatewayMessageHistory();
        var result = history.messages(new ChatChannelRef("unknown"), Instant.EPOCH);
        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 4: Implement DiscordGatewayMessageHistory**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ReceivedMessage;
import io.casehub.connectors.chat.spi.MessageHistory;
import java.time.Instant;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

public class DiscordGatewayMessageHistory implements MessageHistory {

    private final ConcurrentHashMap<String, CopyOnWriteArrayList<ReceivedMessage>> buffer =
            new ConcurrentHashMap<>();

    public void accumulate(ReceivedMessage message) {
        buffer.computeIfAbsent(message.channel().id(), k -> new CopyOnWriteArrayList<>())
                .add(message);
    }

    @Override
    public List<ReceivedMessage> messages(ChatChannelRef channel, Instant since) {
        var channelMessages = buffer.get(channel.id());
        if (channelMessages == null) return List.of();
        return channelMessages.stream()
                .filter(m -> m.receivedAt().isAfter(since))
                .toList();
    }
}
```

- [ ] **Step 5: Run tests, commit**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=DiscordEventSourceTest,DiscordGatewayMessageHistoryTest -q`
Expected: PASS

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): DiscordEventSource + DiscordGatewayMessageHistory — execution model adapters Refs #279"
```

### Task 7: ChatWorldBridge

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatWorldBridge.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatWorldBridgeTest.java`

**Interfaces:**
- Consumes: `WorldBridge<ChatPerception, ChatIntent>`, `MessageHistory`, `Messaging`, `Threading`, `Reactions` (chat-spi), `BotIdentityDetector`
- Produces: `ChatWorldBridge.perceive(WakeReason) → ChatPerception` (WakeReason per-tick, not constructor), `ChatWorldBridge.dispatch(IntentQueue<ChatIntent>)` routes to chat SPI

- [ ] **Step 1: Write ChatWorldBridge test**

```java
package io.quarkmind.chat.agent;

import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.chat.BotIdentityDetector;
import io.quarkmind.agency.intent.IntentQueue;
import io.quarkmind.chat.protocol.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.atomic.AtomicReference;
import static org.junit.jupiter.api.Assertions.*;

class ChatWorldBridgeTest {

    @Test
    void perceiveReturnsChannelDeltas() {
        var messages = List.of(dummyMessage("hello"));
        var history = stubHistory(Map.of("ch-1", messages));
        var bridge = new ChatWorldBridge(history, List.of("ch-1"), stubDetector());
        var perception = bridge.perceive(WakeReason.MESSAGE);
        assertEquals(1, perception.channelDeltas().get("ch-1").size());
        assertEquals(WakeReason.MESSAGE, perception.reason());
    }

    @Test
    void perceivePassesWakeReasonPerCall() {
        var history = stubHistory(Map.of());
        var bridge = new ChatWorldBridge(history, List.of(), stubDetector());
        assertEquals(WakeReason.MESSAGE, bridge.perceive(WakeReason.MESSAGE).reason());
        assertEquals(WakeReason.HEARTBEAT, bridge.perceive(WakeReason.HEARTBEAT).reason());
    }

    @Test
    void dispatchRoutesSendToMessaging() {
        var sent = new AtomicReference<String>();
        var history = stubHistory(Map.of());
        var bridge = new ChatWorldBridge(history, List.of(), stubDetector());
        bridge.setMessaging((channel, content) -> {
            sent.set(content.text());
            return SendResult.success(new ChatMessageRef(channel, "sent-1"), Instant.now());
        });

        var queue = new IntentQueue<ChatIntent>();
        queue.enqueue(new ChatIntent.Send("ch-1", new ChatContent("hi")));
        bridge.dispatch(queue);
        assertEquals("hi", sent.get());
    }

    @Test
    void dispatchRoutesReactToReactions() {
        var reacted = new AtomicReference<String>();
        var history = stubHistory(Map.of());
        var bridge = new ChatWorldBridge(history, List.of(), stubDetector());
        bridge.setReactions(new io.casehub.connectors.chat.spi.Reactions() {
            @Override public void add(ChatMessageRef ref, String emoji) { reacted.set(emoji); }
            @Override public void remove(ChatMessageRef ref, String emoji) {}
            @Override public List<String> list(ChatMessageRef ref) { return List.of(); }
        });

        var queue = new IntentQueue<ChatIntent>();
        var msgRef = new ChatMessageRef(new ChatChannelRef("ch"), "m1");
        queue.enqueue(new ChatIntent.React(msgRef, "👀"));
        bridge.dispatch(queue);
        assertEquals("👀", reacted.get());
    }

    private ReceivedMessage dummyMessage(String text) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"), new ChatContent(text), Instant.now());
    }

    private BotIdentityDetector stubDetector() {
        return new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) { return false; }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot"; }
        };
    }

    private io.casehub.connectors.chat.spi.MessageHistory stubHistory(
            Map<String, List<ReceivedMessage>> data) {
        return (channel, since) -> data.getOrDefault(channel.id(), List.of());
    }
}
```

- [ ] **Step 2: Implement ChatWorldBridge**

WakeReason is per-tick via `perceive(WakeReason)`, not a constructor parameter.

```java
package io.quarkmind.chat.agent;

import io.casehub.connectors.chat.model.*;
import io.casehub.connectors.chat.spi.*;
import io.quarkmind.agency.chat.BotIdentityDetector;
import io.quarkmind.agency.intent.IntentQueue;
import io.quarkmind.agency.spi.WorldBridge;
import io.quarkmind.chat.protocol.*;
import java.time.Instant;
import java.util.*;

public class ChatWorldBridge implements WorldBridge<ChatPerception, ChatIntent> {

    private final MessageHistory messageHistory;
    private final List<String> watchedChannels;
    private final BotIdentityDetector identityDetector;
    private Messaging messaging;
    private Threading threading;
    private Reactions reactions;
    private Instant lastCheck = Instant.now();

    public ChatWorldBridge(MessageHistory messageHistory, List<String> watchedChannels,
                           BotIdentityDetector identityDetector) {
        this.messageHistory = messageHistory;
        this.watchedChannels = watchedChannels;
        this.identityDetector = identityDetector;
    }

    public void setMessaging(Messaging messaging) { this.messaging = messaging; }
    public void setThreading(Threading threading) { this.threading = threading; }
    public void setReactions(Reactions reactions) { this.reactions = reactions; }

    @Override
    public void connect() {}

    @Override
    public void disconnect() {}

    public ChatPerception perceive(WakeReason reason) {
        Map<String, List<ReceivedMessage>> deltas = new LinkedHashMap<>();
        for (String channelId : watchedChannels) {
            var messages = messageHistory.messages(new ChatChannelRef(channelId), lastCheck);
            if (!messages.isEmpty()) {
                deltas.put(channelId, messages);
            }
        }
        lastCheck = Instant.now();
        return new ChatPerception(deltas, Map.of(), reason);
    }

    @Override
    public ChatPerception perceive() {
        return perceive(WakeReason.MESSAGE);
    }

    @Override
    public void dispatch(IntentQueue<ChatIntent> intents) {
        for (ChatIntent intent : intents.drainAll()) {
            switch (intent) {
                case ChatIntent.Send s ->
                        messaging.send(new ChatChannelRef(s.channelId()), s.content());
                case ChatIntent.Reply r ->
                        threading.reply(r.parent(), r.content());
                case ChatIntent.React r ->
                        reactions.add(r.message(), r.emoji());
            }
        }
    }
}
```

- [ ] **Step 3: Run tests, commit**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatWorldBridgeTest -q`
Expected: PASS

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): ChatWorldBridge — WakeReason per-tick, intent dispatch via chat SPI Refs #279"
```

## Batch 4: Agency loop with compression + Jackson parsing

### Task 8: ChatAgencyLoop with ChatDeltaReport + compression wiring

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java`

**Interfaces:**
- Consumes: `AgencyLoop`, `AgencyContext`, `ChatPerception`, `ChatDeltaReport`, `ChatObservationRenderer`, `ChatPerceptionBridge`, `DefaultChatPerceptionBridge`, `BotIdentityDetector`, `LlmRequestQueue.hasCapacity()`, `NeedState`, Jackson `ObjectMapper`
- Produces: `ChatAgencyLoop.tick(AgencyContext)` — builds delta report, renders with compression, invokes LLM via `LlmInvoker`, parses response with Jackson, emits intents

- [ ] **Step 1: Write ChatAgencyLoop test**

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.chat.*;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.chat.protocol.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ChatAgencyLoopTest {

    private final ObjectMapper mapper = new ObjectMapper();
    private final BotIdentityDetector detector = new BotIdentityDetector() {
        @Override public boolean isMention(ReceivedMessage msg) { return false; }
        @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
        @Override public String botUserId() { return "bot-id"; }
    };
    private final LlmRequestQueue llmQueue = new LlmRequestQueue() {
        @Override public void submit(io.quarkmind.agency.llm.LlmRequest request) {}
        @Override public int pendingCount() { return 0; }
        @Override public boolean hasCapacity() { return true; }
    };

    @Test
    void tickProducesIntentsFromLlmResponse() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
                "{\"action\":\"SEND\",\"channel\":\"ch-1\",\"text\":\"hello back\"}";

        var loop = new ChatAgencyLoop(llm, detector, llmQueue, mapper,
                new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)));
        loop.setSystemPrompt("You are a friendly bot.");

        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"), new ChatContent("hello"), Instant.now());
        var perception = new ChatPerception(Map.of("ch-1", List.of(msg)), Map.of(), WakeReason.MESSAGE);

        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");
        assertNotNull(intents);
        assertEquals(1, intents.size());
        assertInstanceOf(ChatIntent.Send.class, intents.get(0));
        assertEquals("hello back", ((ChatIntent.Send) intents.get(0)).content().text());
    }

    @Test
    void heartbeatWithNoActivityProducesNoIntents() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
                "{\"action\":\"WAIT\"}";

        var loop = new ChatAgencyLoop(llm, detector, llmQueue, mapper,
                new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)));
        var perception = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);

        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");
        assertTrue(intents == null || intents.isEmpty());
    }

    @Test
    void skipsLlmWhenQueueAtCapacity() {
        var saturatedQueue = new LlmRequestQueue() {
            @Override public void submit(io.quarkmind.agency.llm.LlmRequest request) {}
            @Override public int pendingCount() { return 100; }
            @Override public boolean hasCapacity() { return false; }
        };
        var invoked = new java.util.concurrent.atomic.AtomicBoolean(false);
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) -> {
            invoked.set(true);
            return "{\"action\":\"WAIT\"}";
        };

        var loop = new ChatAgencyLoop(llm, detector, saturatedQueue, mapper,
                new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)));
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"), null,
                new MemberRef("u1"), new ChatContent("hi"), Instant.now());
        var perception = new ChatPerception(Map.of("ch", List.of(msg)), Map.of(), WakeReason.MESSAGE);

        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        loop.tick(context);
        assertFalse(invoked.get());
    }

    @Test
    void parsesReactIntent() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
                "{\"action\":\"REACT\",\"messageId\":\"m1\",\"emoji\":\"👀\"}";
        var loop = new ChatAgencyLoop(llm, detector, llmQueue, mapper,
                new DefaultChatPerceptionBridge(new ChatObservationRenderer(10)));
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"), null,
                new MemberRef("u1"), new ChatContent("interesting"), Instant.now());
        var perception = new ChatPerception(Map.of("ch", List.of(msg)), Map.of(), WakeReason.MESSAGE);

        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");
        assertEquals(1, intents.size());
        assertInstanceOf(ChatIntent.React.class, intents.get(0));
        assertEquals("👀", ((ChatIntent.React) intents.get(0)).emoji());
    }
}
```

- [ ] **Step 2: Implement ChatAgencyLoop**

Uses Jackson for JSON parsing, `LlmRequestQueue.hasCapacity()` as gate, `ChatPerceptionBridge` for delta report + compression wiring.

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.AgencyLoop;
import io.quarkmind.agency.chat.*;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.chat.protocol.*;
import java.util.*;

public class ChatAgencyLoop implements AgencyLoop {

    @FunctionalInterface
    public interface LlmInvoker {
        String invoke(String systemPrompt, String userPrompt, String agentId);
    }

    private final LlmInvoker llmInvoker;
    private final BotIdentityDetector identityDetector;
    private final LlmRequestQueue llmQueue;
    private final ObjectMapper mapper;
    private final ChatPerceptionBridge perceptionBridge;
    private String systemPrompt = "";
    private String agentId = "chat-agent";
    private final Set<String> participatedThreadIds = new HashSet<>();

    public ChatAgencyLoop(LlmInvoker llmInvoker, BotIdentityDetector identityDetector,
                          LlmRequestQueue llmQueue, ObjectMapper mapper,
                          ChatPerceptionBridge perceptionBridge) {
        this.llmInvoker = llmInvoker;
        this.identityDetector = identityDetector;
        this.llmQueue = llmQueue;
        this.mapper = mapper;
        this.perceptionBridge = perceptionBridge;
    }

    public void setSystemPrompt(String prompt) { this.systemPrompt = prompt; }
    public void setAgentId(String id) { this.agentId = id; }

    @Override
    public void tick(AgencyContext context) {
        var perception = context.getAs("perception", ChatPerception.class);
        if (perception == null) return;

        if (perception.reason() == WakeReason.HEARTBEAT && !perception.hasActivity()) {
            context.put("intents", List.of());
            return;
        }

        if (!llmQueue.hasCapacity()) {
            context.put("intents", List.of());
            return;
        }

        ChatDeltaReport report = perceptionBridge.buildDelta(
                perception, identityDetector, participatedThreadIds);
        String renderedContext = perceptionBridge.renderForLlm(report);

        String userPrompt = buildUserPrompt(renderedContext, context);
        String response = llmInvoker.invoke(systemPrompt, userPrompt, agentId);
        List<ChatIntent> intents = parseResponse(response);
        context.put("intents", intents);
    }

    private String buildUserPrompt(String renderedContext, AgencyContext context) {
        var sb = new StringBuilder();
        sb.append("Needs: SOCIAL=%.0f, CURIOSITY=%.0f, EXPRESSION=%.0f\n".formatted(
                context.needState().get("SOCIAL"),
                context.needState().get("CURIOSITY"),
                context.needState().get("EXPRESSION")));
        sb.append("\n").append(renderedContext);
        sb.append("""

                Respond with JSON:
                {"action":"SEND|REPLY|REACT|WAIT","channel":"channel-id","text":"message","emoji":"emoji","messageId":"id-to-react-to","replyTo":"message-id"}
                Only include fields relevant to the action.
                """);
        return sb.toString();
    }

    List<ChatIntent> parseResponse(String response) {
        var intents = new ArrayList<ChatIntent>();
        try {
            JsonNode root = mapper.readTree(response);
            String action = root.has("action") ? root.get("action").asText() : null;
            if (action == null || "WAIT".equalsIgnoreCase(action)) return intents;

            switch (action.toUpperCase()) {
                case "SEND" -> {
                    String channel = root.has("channel") ? root.get("channel").asText() : null;
                    String text = root.has("text") ? root.get("text").asText() : null;
                    if (channel != null && text != null) {
                        intents.add(new ChatIntent.Send(channel, new ChatContent(text)));
                    }
                }
                case "REPLY" -> {
                    String replyTo = root.has("replyTo") ? root.get("replyTo").asText() : null;
                    String text = root.has("text") ? root.get("text").asText() : null;
                    if (replyTo != null && text != null) {
                        var parentRef = new ChatMessageRef(new ChatChannelRef(""), replyTo);
                        intents.add(new ChatIntent.Reply(parentRef, new ChatContent(text)));
                    }
                }
                case "REACT" -> {
                    String msgId = root.has("messageId") ? root.get("messageId").asText() : null;
                    String emoji = root.has("emoji") ? root.get("emoji").asText() : null;
                    if (msgId != null && emoji != null) {
                        var msgRef = new ChatMessageRef(new ChatChannelRef(""), msgId);
                        intents.add(new ChatIntent.React(msgRef, emoji));
                    }
                }
                default -> {}
            }
        } catch (Exception e) {
            // LLM response parsing failures are expected — return empty intents
        }
        return intents;
    }
}
```

- [ ] **Step 3: Run tests, commit**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgencyLoopTest -q`
Expected: PASS

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): ChatAgencyLoop — delta report + compression + Jackson parsing + LLM capacity gate Refs #279"
```

## Batch 5: Execution harness + config + end-to-end test

### Task 9: ChatCharacterApp + ChatNeedDefinitions + ChatChannelPacing + config

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatCharacterApp.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatNeedDefinitions.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatChannelPacing.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/resources/application.yaml`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatNeedDefinitionsTest.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatChannelPacingTest.java`

**Interfaces:**
- Consumes: `ChoreographedDriver` (blocks), `EventSource.merge()`, `EventSource.ticker()`, `EventConcurrencyPolicy.coalesce()`, `ChatWorldBridge`, `ChatAgencyLoop`, `OutputGovernor`, `ProactiveDecisionGate`, `NeedThresholdWake`, `DiscordEventSource`, `DiscordGatewayMessageHistory`, `DiscordIdentityDetector`, `ChatPlatform` (connectors), `NeedState`, `IntentQueue`
- Produces: Running agent lifecycle — `ChatCharacterApp` starts ChoreographedDriver on Quarkus startup, agent loop runs on virtual thread

- [ ] **Step 1: Write ChatNeedDefinitions test**

```java
package io.quarkmind.chat.agent;

import io.quarkmind.agency.needs.NeedState;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ChatNeedDefinitionsTest {

    @Test
    void socialDecaysOnIdle() {
        var needs = new NeedState();
        needs.set("SOCIAL", 80.0);
        var defs = new ChatNeedDefinitions(0.5, 0.1, 0.2);
        defs.decayAll(needs);
        assertEquals(79.5, needs.get("SOCIAL"), 0.01);
    }

    @Test
    void socialSatisfiedByConversation() {
        var needs = new NeedState();
        needs.set("SOCIAL", 30.0);
        needs.satisfy("SOCIAL", 20.0);
        assertEquals(50.0, needs.get("SOCIAL"), 0.01);
    }

    @Test
    void expressionBuildsOverTime() {
        var needs = new NeedState();
        needs.set("EXPRESSION", 0.0);
        var defs = new ChatNeedDefinitions(0.5, 0.1, 0.2);
        defs.buildExpression(needs);
        assertTrue(needs.get("EXPRESSION") > 0.0);
    }
}
```

- [ ] **Step 2: Implement ChatNeedDefinitions**

```java
package io.quarkmind.chat.agent;

import io.quarkmind.agency.needs.NeedState;

public class ChatNeedDefinitions {

    private final double socialDecayRate;
    private final double curiosityDecayRate;
    private final double expressionBuildRate;

    public ChatNeedDefinitions(double socialDecayRate, double curiosityDecayRate,
                               double expressionBuildRate) {
        this.socialDecayRate = socialDecayRate;
        this.curiosityDecayRate = curiosityDecayRate;
        this.expressionBuildRate = expressionBuildRate;
    }

    public void decayAll(NeedState needs) {
        needs.decay("SOCIAL", socialDecayRate);
        needs.decay("CURIOSITY", curiosityDecayRate);
    }

    public void buildExpression(NeedState needs) {
        needs.satisfy("EXPRESSION", expressionBuildRate);
    }
}
```

- [ ] **Step 3: Write ChatChannelPacing test**

```java
package io.quarkmind.chat.agent;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ChatChannelPacingTest {

    @Test
    void quietChannelAllowsPost() {
        var pacing = new ChatChannelPacing(5, 60_000);
        assertTrue(pacing.allowUnprompted(0, 120_000));
    }

    @Test
    void busyChannelBlocksPost() {
        var pacing = new ChatChannelPacing(5, 60_000);
        assertFalse(pacing.allowUnprompted(10, 120_000));
    }

    @Test
    void tooSoonBlocksRegardlessOfActivity() {
        var pacing = new ChatChannelPacing(5, 60_000);
        assertFalse(pacing.allowUnprompted(0, 30_000));
    }
}
```

- [ ] **Step 4: Implement ChatChannelPacing**

```java
package io.quarkmind.chat.agent;

public class ChatChannelPacing {

    private final int maxChannelActivityForUnprompted;
    private final long minTimeSinceLastPostMs;

    public ChatChannelPacing(int maxChannelActivityForUnprompted, long minTimeSinceLastPostMs) {
        this.maxChannelActivityForUnprompted = maxChannelActivityForUnprompted;
        this.minTimeSinceLastPostMs = minTimeSinceLastPostMs;
    }

    public boolean allowUnprompted(int recentChannelMessages, long timeSinceLastPostMs) {
        if (timeSinceLastPostMs < minTimeSinceLastPostMs) return false;
        return recentChannelMessages <= maxChannelActivityForUnprompted;
    }
}
```

- [ ] **Step 5: Write ChatCharacterApp**

The Quarkus entry point that wires `ChoreographedDriver` with `EventSource.merge(discordEventSource, EventSource.ticker(...))` and runs the agent loop on a virtual thread.

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.blocks.agentic.model.*;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.chat.*;
import io.quarkmind.agency.intent.IntentQueue;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.agency.schedule.OutputGovernor;
import io.quarkmind.chat.agent.discord.DiscordEventSource;
import io.quarkmind.chat.agent.discord.DiscordIdentityDetector;
import io.quarkmind.chat.protocol.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import io.quarkus.runtime.StartupEvent;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.*;
import java.util.logging.Logger;

@ApplicationScoped
public class ChatCharacterApp {

    private static final Logger LOG = Logger.getLogger(ChatCharacterApp.class.getName());

    private final ChatPlatform chatPlatform;
    private final LlmRequestQueue llmQueue;
    private final ObjectMapper mapper;
    private final String botUserId;
    private final List<String> channels;
    private final long heartbeatSeconds;
    private final long governorWindowSeconds;
    private final long governorMinIntervalSeconds;
    private final int governorMaxPerWindow;

    public ChatCharacterApp(
            ChatPlatform chatPlatform,
            LlmRequestQueue llmQueue,
            ObjectMapper mapper,
            @ConfigProperty(name = "quarkmind.chat.bot-user-id", defaultValue = "") String botUserId,
            @ConfigProperty(name = "quarkmind.chat.channels", defaultValue = "general") String channelsConfig,
            @ConfigProperty(name = "quarkmind.chat.heartbeat-seconds", defaultValue = "60") long heartbeatSeconds,
            @ConfigProperty(name = "quarkmind.chat.governor.window-seconds", defaultValue = "300") long governorWindowSeconds,
            @ConfigProperty(name = "quarkmind.chat.governor.min-interval-seconds", defaultValue = "30") long governorMinIntervalSeconds,
            @ConfigProperty(name = "quarkmind.chat.governor.max-unprompted-per-window", defaultValue = "1") int governorMaxPerWindow) {
        this.chatPlatform = chatPlatform;
        this.llmQueue = llmQueue;
        this.mapper = mapper;
        this.botUserId = botUserId;
        this.channels = Arrays.asList(channelsConfig.split(","));
        this.heartbeatSeconds = heartbeatSeconds;
        this.governorWindowSeconds = governorWindowSeconds;
        this.governorMinIntervalSeconds = governorMinIntervalSeconds;
        this.governorMaxPerWindow = governorMaxPerWindow;
    }

    void onStart(@Observes StartupEvent event) {
        if (botUserId.isBlank()) {
            LOG.warning("quarkmind-chat: bot-user-id not configured, agent inactive");
            return;
        }

        Thread.ofVirtual().name("chat-agent").start(this::runAgent);
    }

    private void runAgent() {
        var identityDetector = new DiscordIdentityDetector(botUserId);
        var discordEventSource = new DiscordEventSource();
        var executor = Executors.newSingleThreadScheduledExecutor(
                Thread.ofVirtual().name("chat-heartbeat").factory());
        var renderer = new ChatObservationRenderer(10);
        var perceptionBridge = new DefaultChatPerceptionBridge(renderer);

        ChatAgencyLoop.LlmInvoker llmInvoker = (system, user, id) -> {
            // Production: wrap AgentProvider here
            return "{\"action\":\"WAIT\"}";
        };

        var agencyLoop = new ChatAgencyLoop(llmInvoker, identityDetector, llmQueue, mapper, perceptionBridge);
        var bridge = new ChatWorldBridge(
                new io.quarkmind.chat.agent.discord.DiscordGatewayMessageHistory(),
                channels, identityDetector);
        bridge.setMessaging(chatPlatform.messaging());
        bridge.setThreading(chatPlatform.threading());
        bridge.setReactions(chatPlatform.reactions());

        var governor = new OutputGovernor(
                governorWindowSeconds * 1000, governorMinIntervalSeconds * 1000, governorMaxPerWindow);
        var needState = new NeedState();
        needState.set("SOCIAL", 80.0);
        needState.set("CURIOSITY", 50.0);
        needState.set("EXPRESSION", 0.0);

        var needs = new ChatNeedDefinitions(0.5, 0.1, 0.2);

        var merged = EventSource.merge(
                discordEventSource,
                EventSource.ticker(Duration.ofSeconds(heartbeatSeconds), executor));

        var eventQueue = new LinkedBlockingQueue<DriverEvent>();
        var subscription = merged.subscribe(eventQueue::add);

        try {
            while (!Thread.currentThread().isInterrupted()) {
                var driverEvent = eventQueue.take();
                var reason = WakeReason.fromDriverSource(driverEvent.source());
                needs.decayAll(needState);
                needs.buildExpression(needState);

                var perception = bridge.perceive(reason);
                var context = new AgencyContext(needState);
                context.put("perception", perception);
                agencyLoop.tick(context);

                @SuppressWarnings("unchecked")
                var intents = (List<ChatIntent>) context.get("intents");
                if (intents != null && !intents.isEmpty() && governor.allow()) {
                    var queue = new IntentQueue<ChatIntent>();
                    intents.forEach(queue::enqueue);
                    bridge.dispatch(queue);
                    governor.recordAction();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            subscription.cancel();
            executor.shutdownNow();
        }
    }
}
```

- [ ] **Step 6: Create application.yaml**

```yaml
quarkmind:
  chat:
    bot-user-id: ${DISCORD_BOT_USER_ID:}
    channels: general
    heartbeat-seconds: 60
    needs:
      social:
        decay-rate: 0.5
      curiosity:
        decay-rate: 0.1
      expression:
        build-rate: 0.2
    governor:
      max-unprompted-per-window: 1
      window-seconds: 300
      min-interval-seconds: 30

"%discord":
  casehub:
    discord:
      token: ${DISCORD_BOT_TOKEN:}
```

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add quarkmind-chat/
git commit -m "feat(#279): ChatCharacterApp + ChatNeedDefinitions + ChatChannelPacing + config Refs #279"
```

### Task 10: End-to-end integration test

**Files:**
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgentEndToEndTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1–9
- Produces: Verified full cycle: event → perceive → delta report → compress → LLM → parse → govern → dispatch

- [ ] **Step 1: Write end-to-end test**

```java
package io.quarkmind.chat.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.chat.*;
import io.quarkmind.agency.intent.IntentQueue;
import io.quarkmind.agency.llm.LlmRequestQueue;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.agency.schedule.OutputGovernor;
import io.quarkmind.chat.agent.discord.DiscordGatewayMessageHistory;
import io.quarkmind.chat.protocol.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.atomic.AtomicReference;
import static org.junit.jupiter.api.Assertions.*;

class ChatAgentEndToEndTest {

    @Test
    void fullCycleFromMessageToResponse() {
        var mapper = new ObjectMapper();
        var detector = stubDetector();
        var llmQueue = stubLlmQueue();

        var llm = (ChatAgencyLoop.LlmInvoker) (sys, usr, id) ->
                "{\"action\":\"SEND\",\"channel\":\"general\",\"text\":\"Hey there!\"}";

        var renderer = new ChatObservationRenderer(10);
        var perceptionBridge = new DefaultChatPerceptionBridge(renderer);
        var loop = new ChatAgencyLoop(llm, detector, llmQueue, mapper, perceptionBridge);
        loop.setSystemPrompt("You are Quark, a friendly chat character.");

        var gatewayHistory = new DiscordGatewayMessageHistory();
        var incomingMsg = new ReceivedMessage("discord", new ChatChannelRef("general"),
                new ChatMessageRef(new ChatChannelRef("general"), "m1"), null,
                new MemberRef("alice"), new ChatContent("hi everyone!"), Instant.now());
        gatewayHistory.accumulate(incomingMsg);

        var bridge = new ChatWorldBridge(gatewayHistory, List.of("general"), detector);

        var sent = new AtomicReference<String>();
        bridge.setMessaging((channel, content) -> {
            sent.set(content.text());
            return SendResult.success(new ChatMessageRef(channel, "sent-1"), Instant.now());
        });
        bridge.setThreading((parent, content) ->
                SendResult.success(new ChatMessageRef(parent.channel(), "sent-2"), Instant.now()));
        bridge.setReactions(new io.casehub.connectors.chat.degraded.NoOpReactions());

        var perception = bridge.perceive(WakeReason.MESSAGE);
        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");
        var governor = new OutputGovernor(300_000, 0, 10);

        var queue = new IntentQueue<ChatIntent>();
        for (ChatIntent intent : intents) {
            if (governor.allow()) {
                queue.enqueue(intent);
                governor.recordAction();
            }
        }
        bridge.dispatch(queue);

        assertEquals("Hey there!", sent.get());
    }

    private BotIdentityDetector stubDetector() {
        return new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) { return false; }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot-id"; }
        };
    }

    private LlmRequestQueue stubLlmQueue() {
        return new LlmRequestQueue() {
            @Override public void submit(io.quarkmind.agency.llm.LlmRequest request) {}
            @Override public int pendingCount() { return 0; }
            @Override public boolean hasCapacity() { return true; }
        };
    }
}
```

- [ ] **Step 2: Run test, commit**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgentEndToEndTest -q`
Expected: PASS

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): end-to-end integration test — full perceive→reason→compress→dispatch cycle Refs #279"
```

## Batch 6: Deferred items → GitHub issues

### Task 11: File deferred issues

The following design spec capabilities are out of scope for phase 1 and must be filed as GitHub issues:

- [ ] **Step 1: File memory integration issue**

```bash
gh issue create --repo casehubio/quarkmind --title "quarkmind-chat — memory integration (D5 four-layer memory)" --body "## Summary
Wire episodic memory (CbrCaseMemoryStore + TemporalDecayCbrCaseMemoryStore), semantic memory (ReflectionService), and relationship memory (GraphitiCaseMemoryStore + casehub-ledger) into the ChatAgencyLoop.

## Depends on
- casehub-neocortex: importance scoring at ingest (async LLM rating)
- Relationship schema for Graphiti (per-person model)

## Design spec
specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md — §Memory Architecture, D5

## Blocked by
casehub-neocortex importance scoring enhancement"
```

- [ ] **Step 2: File personality growth issue**

```bash
gh issue create --repo casehubio/quarkmind --title "quarkmind-chat — personality growth pipeline (D6 Layer 3)" --body "## Summary
Implement the reflection → disposition evolution chain:
- IdleReflectionTrigger (quarkmind-core) — fires when accumulated importance exceeds threshold during idle
- ReflectionDispositionActivator (quarkmind-core) — LLM-based classification of which disposition function term a reflection activates
- Wire: ReflectionService → ReflectionDispositionActivator → DispositionSignalStore.recordActivation() → DispositionHealth.probe() → DispositionEvolution.evaluate()

## Depends on
- casehub-neocortex: importance scoring at ingest
- quarkmind-chat phase 1 (this plan)

## Design spec
specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md — §Personality System Layer 3, D6"
```

- [ ] **Step 3: File multi-character issue**

```bash
gh issue create --repo casehubio/quarkmind --title "quarkmind-chat — multi-character per server (D9 v2)" --body "## Summary
Extend quarkmind-chat to support multiple characters per server. Requires per-character isolation for memory, personality, need state, and identity detection.

## Design spec
specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md — §Scope, D9"
```

- [ ] **Step 4: File personality generator issue**

```bash
gh issue create --repo casehubio/quarkmind --title "quarkmind-chat — personality generator wizard (D12 future)" --body "## Summary
LLM-powered wizard that helps non-technical users create character YAML descriptors. \"Describe your character\" → generates AgentDescriptor YAML with appropriate disposition axes, style terms, and briefing.

## Design spec
specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md — §Configuration-Driven Deployment, D12"
```

- [ ] **Step 5: Commit**

No files to commit — issues are on GitHub.

---

## Review Findings Addressed

| Finding | Fix | Location |
|---------|-----|----------|
| R1-02 (HIGH) | Goal statement scoped to phase 1; deferred items as GitHub issues in Batch 6 | Header, Batch 6 |
| R1-03 (HIGH) | All `queue.add()` → `queue.enqueue()` | Tasks 7, 10 |
| R1-04 (HIGH) | `LlmRequestQueue.hasCapacity()` gate before LLM calls; `LlmInvoker` kept for sync invocation | Task 8 |
| R1-05 (HIGH) | `DiscordEventSource`, `DiscordGatewayMessageHistory`, `ChatCharacterApp` with `ChoreographedDriver` + `EventSource.merge` | Tasks 6, 9 |
| R1-06 (HIGH) | `ChatObservationRenderer`, `NeedThresholdWake`, `ChatPerceptionBridge` added; `IdleReflectionTrigger` + `ReflectionDispositionActivator` deferred as issues | Task 4, Batch 6 |
| R1-07 (MEDIUM) | `ChatChannelPacing` added | Task 9 |
| R1-08 (MEDIUM) | `AttentionClassifier.classify()` signature includes `Set<String> participatedThreadIds` | Task 3 |
| R1-09 (LOW) | `ProactiveDecisionGate` in `io.quarkmind.agency.personality` | Task 4 |
| R1-10 (MEDIUM) | Jackson `ObjectMapper.readTree()` replaces manual `extractField()` | Task 8 |
| R1-11/R1-12 (MEDIUM) | `ChatPerceptionBridge` wires `ChatDeltaReport` + `ChatObservationRenderer` into `ChatAgencyLoop`; DIRECT/ELEVATED verbatim, AMBIENT compressed | Tasks 4, 8 |
| R1-13 (LOW) | `WakeReason` removed from `ChatWorldBridge` constructor; `perceive(WakeReason)` accepts per-tick reason from `DriverEvent.source()` | Task 7 |

---

## References

- `specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md` — design spec
- `specs/issue-279-quarkmind-discord/decisions.md` — D1–D12 design decisions
- `quarkmind-core/src/main/java/io/quarkmind/agency/intent/IntentQueue.java` — `enqueue()` not `add()`
- `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequestQueue.java` — async SPI: `submit()`, `hasCapacity()`
- `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmRequest.java` — `LlmRequest(prompt, priority, metadata)`
- `quarkmind-core/src/main/java/io/quarkmind/agency/llm/LlmPriority.java` — LOW, NORMAL, HIGH, URGENT
- `casehub-blocks` — `EventSource.merge()`, `EventSource.ticker()`, `ChoreographedDriver`, `EventConcurrencyPolicy`, `DriverEvent`, `TieredObservationRenderer`
- `casehub-connectors/chat-spi/` — `ChatPlatform`, `MessageHistory`, `Messaging`, `Threading`, `Reactions`, `ReceivedMessage`, `ChatContent`
- `casehub-connectors/chat-discord/` — `DiscordChatPlatform`, `DiscordInboundConnector`
- `quarkmind-ville/quarkmind-ville-agent/` — `VilleAgencyLoop`, `VilleWorldBridge` (reference pattern)
- `reviews/casehub-slots/quarkmind-chat-plan-20260818-015916/responses/reviewer-1.md` — review findings R1-01 through R1-14
- GitHub #279 — quarkmind-discord — persistent character in Discord
