# QuarkMind Chat — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #279 — quarkmind-discord — persistent character in Discord
**Issue group:** #279

**Goal:** Build quarkmind-chat — a platform-agnostic harness for advanced AI chat bots with personality, memory, and autonomous behavior. Discord is the first target platform.

**Architecture:** Two sub-modules (quarkmind-chat-protocol, quarkmind-chat-agent) under a quarkmind-chat parent. Platform-agnostic agency loop using quarkmind-core SPIs. Chat-specific abstractions (delta reports, attention classification, observation rendering) added to quarkmind-core. Discord adapter layer uses casehub-connectors chat-spi + discord modules. Direct LLM agency loop (no CaseEngine), needs-driven autonomous behavior, configuration-driven character deployment.

**Tech Stack:** Java 21+, Quarkus, casehub-connectors (chat-spi, chat-discord, discord), casehub-blocks (TieredObservationRenderer, EventSource, ChoreographedDriver), casehub-eidos-api, casehub-neocortex-api, casehub-ledger-api, quarkmind-core agency framework

## Global Constraints

- All code follows quarkmind-core package conventions (`io.quarkmind.agency.*` for core, `io.quarkmind.chat.*` for chat module)
- No CaseEngine dependency — direct LLM via `LlmRequestQueue`
- Platform-specific code lives in sub-packages (`discord/`, `slack/`, etc.), never in the main package
- All tuning knobs externalized as Quarkus config properties — no hardcoded constants
- Tests run without a live Discord connection or LLM endpoint
- `ChatPlatform` SPI is the abstraction boundary — no direct Discord API calls from agent code
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

- [ ] **Step 3: Create agent sub-module POM**

`quarkmind-chat/quarkmind-chat-agent/pom.xml` — Quarkus application:

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
            <groupId>io.casehub</groupId>
            <artifactId>casehub-neocortex-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
    </dependencies>
</project>
```

Note: exact artifact IDs for casehub dependencies must be verified against the parent POM's `<dependencyManagement>` section. Check `quarkmind/pom.xml` and cross-reference with the connectors, blocks, eidos, neocortex, and ledger POMs.

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
- Consumes: `io.quarkmind.agency.intent.Intent` (marker interface from quarkmind-core), `io.casehub.connectors.chat.model.ChatContent`, `ChatChannelRef`, `ChatMessageRef`, `MemberRef`, `ReceivedMessage`, `PresenceStatus`
- Produces: `ChatIntent` (sealed: Send, Reply, React), `ChatPerception`, `WakeReason`

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
        var content = new ChatContent("hello", null, java.util.List.of(), java.util.List.of());
        var intent = new ChatIntent.Send("channel-1", content);
        assertEquals("channel-1", intent.channelId());
        assertEquals("hello", intent.content().text());
    }

    @Test
    void replyCreatesWithParentAndContent() {
        var channel = new ChatChannelRef("ch-1");
        var parent = new ChatMessageRef(channel, "msg-1");
        var content = new ChatContent("reply text", null, java.util.List.of(), java.util.List.of());
        var intent = new ChatIntent.Reply(parent, content);
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
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"),
                new ChatContent("hello", null, List.of(), List.of()),
                Instant.now());
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
    void hasDirectAddress_returnsTrueWhenMentioned() {
        var perception = new ChatPerception(Map.of(), Map.of(), WakeReason.MESSAGE);
        assertFalse(perception.hasActivity());
        var withDeltas = new ChatPerception(
                Map.of("ch", List.of(dummyMessage())), Map.of(), WakeReason.MESSAGE);
        assertTrue(withDeltas.hasActivity());
    }

    private ReceivedMessage dummyMessage() {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"), null,
                new MemberRef("u1"),
                new ChatContent("hi", null, List.of(), List.of()),
                Instant.now());
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
- Produces: `AttentionClassifier.classify(List<ReceivedMessage>, BotIdentityDetector) → List<ClassifiedMessage>`, `ChatDeltaReport.build(Map<String, List<ReceivedMessage>>, BotIdentityDetector, Set<String> participatedThreads) → ChatDeltaReport`

- [ ] **Step 1: Write AttentionPriority enum**

```java
package io.quarkmind.agency.chat;

public enum AttentionPriority {
    DIRECT,    // replies to bot, @mentions
    ELEVATED,  // threads bot previously participated in
    AMBIENT    // general channel chatter
}
```

- [ ] **Step 2: Write BotIdentityDetector SPI**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;

public interface BotIdentityDetector {
    boolean isMention(ReceivedMessage message);
    boolean isReplyToBot(ReceivedMessage message);
    String botUserId();
}
```

- [ ] **Step 3: Write ClassifiedMessage record**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;

public record ClassifiedMessage(ReceivedMessage message, AttentionPriority priority) {}
```

- [ ] **Step 4: Write AttentionClassifier test**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
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
        var classified = AttentionClassifier.classify(List.of(msg), detector, java.util.Set.of());
        assertEquals(AttentionPriority.DIRECT, classified.get(0).priority());
    }

    @Test
    void replyToBotClassifiedAsDirect() {
        var parent = new ChatMessageRef(new ChatChannelRef("ch"), "bot-msg-1");
        var msg = msg("sure thing", parent);
        var classified = AttentionClassifier.classify(List.of(msg), detector, java.util.Set.of());
        assertEquals(AttentionPriority.DIRECT, classified.get(0).priority());
    }

    @Test
    void messageInParticipatedThreadClassifiedAsElevated() {
        var parent = new ChatMessageRef(new ChatChannelRef("ch"), "thread-1");
        var msg = msg("continuing discussion", parent);
        var classified = AttentionClassifier.classify(
                List.of(msg), detector, java.util.Set.of("thread-1"));
        assertEquals(AttentionPriority.ELEVATED, classified.get(0).priority());
    }

    @Test
    void ordinaryMessageClassifiedAsAmbient() {
        var msg = msg("hey everyone", null);
        var classified = AttentionClassifier.classify(List.of(msg), detector, java.util.Set.of());
        assertEquals(AttentionPriority.AMBIENT, classified.get(0).priority());
    }

    private ReceivedMessage msg(String text, ChatMessageRef parent) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m-" + text.hashCode()),
                parent, new MemberRef("user-1"),
                new ChatContent(text, null, List.of(), List.of()), Instant.now());
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest -q`
Expected: FAIL — `AttentionClassifier` does not exist

- [ ] **Step 6: Implement AttentionClassifier**

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

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest -q`
Expected: PASS

- [ ] **Step 8: Write ConversationThread record**

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

- [ ] **Step 9: Write ChatDeltaReport test**

Test that the delta report reconstructs threads from `parentRef` chains and classifies new vs continuing:

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
                Map.of("ch", List.of(root, reply, standalone)),
                detector, Set.of());

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
        var threads = report.threads("ch");
        assertEquals(AttentionPriority.DIRECT, threads.get(0).highestPriority());
    }

    @Test
    void emptyDeltaProducesEmptyReport() {
        var report = ChatDeltaReport.build(Map.of(), detector, Set.of());
        assertTrue(report.allChannels().isEmpty());
    }

    private ReceivedMessage msg(String text, ChatMessageRef parent, String id) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), id),
                parent, new MemberRef("user-1"),
                new ChatContent(text, null, List.of(), List.of()), Instant.now());
    }

    private ChatMessageRef ref(String id) {
        return new ChatMessageRef(new ChatChannelRef("ch"), id);
    }
}
```

- [ ] **Step 10: Implement ChatDeltaReport**

```java
package io.quarkmind.agency.chat;

import io.casehub.connectors.chat.model.ReceivedMessage;
import java.util.*;
import java.util.stream.Collectors;

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

    public static ChatDeltaReport build(
            Map<String, List<ReceivedMessage>> channelDeltas,
            BotIdentityDetector detector,
            Set<String> participatedThreadIds) {

        var result = new LinkedHashMap<String, List<ConversationThread>>();
        for (var entry : channelDeltas.entrySet()) {
            String channelId = entry.getKey();
            List<ReceivedMessage> messages = entry.getValue();
            List<ClassifiedMessage> classified =
                    AttentionClassifier.classify(messages, detector, participatedThreadIds);

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

- [ ] **Step 11: Run all tests**

Run: `mvn test -pl quarkmind-core -Dtest=AttentionClassifierTest,ChatDeltaReportTest -q`
Expected: PASS

- [ ] **Step 12: Commit**

```bash
git add quarkmind-core/src/
git commit -m "feat(#279): AttentionClassifier + ChatDeltaReport — chat conversation analysis Refs #279"
```

### Task 4: OutputGovernor + ProactiveDecisionGate

**Files:**
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/OutputGovernor.java`
- Create: `quarkmind-core/src/main/java/io/quarkmind/agency/schedule/ProactiveDecisionGate.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/OutputGovernorTest.java`
- Test: `quarkmind-core/src/test/java/io/quarkmind/agency/schedule/ProactiveDecisionGateTest.java`

**Interfaces:**
- Consumes: `NeedState` from quarkmind-core
- Produces: `OutputGovernor.allow() → boolean` (rate limit check), `ProactiveDecisionGate.shouldAct(long timeSinceLastPost, int channelActivityCount, boolean othersTyping) → boolean`

- [ ] **Step 1: Write OutputGovernor test**

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

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl quarkmind-core -Dtest=OutputGovernorTest -q`
Expected: FAIL

- [ ] **Step 3: Implement OutputGovernor**

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

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl quarkmind-core -Dtest=OutputGovernorTest -q`
Expected: PASS

- [ ] **Step 5: Write ProactiveDecisionGate test**

```java
package io.quarkmind.agency.schedule;

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

- [ ] **Step 6: Implement ProactiveDecisionGate**

```java
package io.quarkmind.agency.schedule;

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

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl quarkmind-core -Dtest=OutputGovernorTest,ProactiveDecisionGateTest -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add quarkmind-core/src/
git commit -m "feat(#279): OutputGovernor + ProactiveDecisionGate — rate governing Refs #279"
```

## Batch 3: ChatWorldBridge + Discord adapter + agency loop

### Task 5: Discord adapters — DiscordIdentityDetector

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/discord/DiscordIdentityDetector.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/discord/DiscordIdentityDetectorTest.java`

**Interfaces:**
- Consumes: `BotIdentityDetector` (SPI from quarkmind-core), `ReceivedMessage` from connectors
- Produces: `DiscordIdentityDetector implements BotIdentityDetector` — detects Discord @mentions and replies to bot

- [ ] **Step 1: Write DiscordIdentityDetector test**

```java
package io.quarkmind.chat.agent.discord;

import io.casehub.connectors.chat.model.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class DiscordIdentityDetectorTest {

    private final DiscordIdentityDetector detector = new DiscordIdentityDetector("12345");

    @Test
    void detectsMentionWithDiscordSyntax() {
        var msg = msg("hey <@12345> what's up");
        assertTrue(detector.isMention(msg));
    }

    @Test
    void doesNotDetectMentionOfOtherUser() {
        var msg = msg("hey <@99999> what's up");
        assertFalse(detector.isMention(msg));
    }

    @Test
    void detectsReplyToBot() {
        var botParent = new ChatMessageRef(new ChatChannelRef("ch"), "bot-sent-msg");
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"),
                botParent, new MemberRef("user-1"),
                new ChatContent("replying", null, List.of(), List.of()), Instant.now());
        detector.recordBotMessage("bot-sent-msg");
        assertTrue(detector.isReplyToBot(msg));
    }

    @Test
    void doesNotDetectReplyToOtherMessage() {
        var otherParent = new ChatMessageRef(new ChatChannelRef("ch"), "other-msg");
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"),
                otherParent, new MemberRef("user-1"),
                new ChatContent("replying", null, List.of(), List.of()), Instant.now());
        assertFalse(detector.isReplyToBot(msg));
    }

    private ReceivedMessage msg(String text) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch"),
                new ChatMessageRef(new ChatChannelRef("ch"), "m1"), null,
                new MemberRef("user-1"),
                new ChatContent(text, null, List.of(), List.of()), Instant.now());
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

- [ ] **Step 3: Run test**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=DiscordIdentityDetectorTest -q`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): DiscordIdentityDetector — @mention and reply-to-bot detection Refs #279"
```

### Task 6: ChatWorldBridge + ChatAgencyLoop skeleton

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatWorldBridge.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatAgencyLoop.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatWorldBridgeTest.java`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgencyLoopTest.java`

**Interfaces:**
- Consumes: `WorldBridge<ChatPerception, ChatIntent>`, `AgencyLoop`, `AgencyContext`, `ChatPlatform` SPI, `ChatDeltaReport`, `AttentionClassifier`, `BotIdentityDetector`, `OutputGovernor`, `ProactiveDecisionGate`, `LlmRequestQueue`, `NeedState`
- Produces: `ChatWorldBridge` (perceive → ChatPerception, dispatch → ChatIntent via ChatPlatform), `ChatAgencyLoop` (tick → perceive/reason/act cycle)

- [ ] **Step 1: Write ChatWorldBridge test**

Tests that `perceive()` builds `ChatPerception` from `MessageHistory` delta, and `dispatch()` routes intents to `ChatPlatform`:

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
        var history = new StubMessageHistory(messages);
        var bridge = new ChatWorldBridge(
                history, List.of("ch-1"), stubDetector(), WakeReason.MESSAGE);
        var perception = bridge.perceive();
        assertEquals(1, perception.channelDeltas().get("ch-1").size());
    }

    @Test
    void dispatchRoutesSendToMessaging() {
        var sent = new AtomicReference<String>();
        var messaging = (io.casehub.connectors.chat.spi.Messaging)
                (channel, content) -> {
                    sent.set(content.text());
                    return SendResult.success(
                            new ChatMessageRef(channel, "sent-1"), Instant.now());
                };
        var bridge = new ChatWorldBridge(
                new StubMessageHistory(List.of()), List.of(), stubDetector(), WakeReason.MESSAGE);
        bridge.setMessaging(messaging);
        var queue = new IntentQueue<ChatIntent>();
        queue.add(new ChatIntent.Send("ch-1",
                new ChatContent("hi", null, List.of(), List.of())));
        bridge.dispatch(queue);
        assertEquals("hi", sent.get());
    }

    private ReceivedMessage dummyMessage(String text) {
        return new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"),
                new ChatContent(text, null, List.of(), List.of()), Instant.now());
    }

    private BotIdentityDetector stubDetector() {
        return new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) { return false; }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot"; }
        };
    }

    private static class StubMessageHistory
            implements io.casehub.connectors.chat.spi.MessageHistory {
        private final List<ReceivedMessage> messages;
        StubMessageHistory(List<ReceivedMessage> messages) { this.messages = messages; }
        @Override
        public List<ReceivedMessage> messages(ChatChannelRef channel, Instant since) {
            return messages;
        }
    }
}
```

- [ ] **Step 2: Implement ChatWorldBridge**

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
    private WakeReason currentWakeReason;

    public ChatWorldBridge(MessageHistory messageHistory, List<String> watchedChannels,
                           BotIdentityDetector identityDetector, WakeReason initialReason) {
        this.messageHistory = messageHistory;
        this.watchedChannels = watchedChannels;
        this.identityDetector = identityDetector;
        this.currentWakeReason = initialReason;
    }

    public void setMessaging(Messaging messaging) { this.messaging = messaging; }
    public void setThreading(Threading threading) { this.threading = threading; }
    public void setReactions(Reactions reactions) { this.reactions = reactions; }
    public void setWakeReason(WakeReason reason) { this.currentWakeReason = reason; }

    @Override
    public void connect() {}

    @Override
    public void disconnect() {}

    @Override
    public ChatPerception perceive() {
        Map<String, List<ReceivedMessage>> deltas = new LinkedHashMap<>();
        for (String channelId : watchedChannels) {
            var messages = messageHistory.messages(new ChatChannelRef(channelId), lastCheck);
            if (!messages.isEmpty()) {
                deltas.put(channelId, messages);
            }
        }
        lastCheck = Instant.now();
        return new ChatPerception(deltas, Map.of(), currentWakeReason);
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

- [ ] **Step 3: Write ChatAgencyLoop test**

Verify the loop calls LLM, parses response, and produces intents:

```java
package io.quarkmind.chat.agent;

import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.chat.protocol.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ChatAgencyLoopTest {

    @Test
    void tickProducesIntentsFromLlmResponse() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
                "{\"action\":\"SEND\",\"channel\":\"ch-1\",\"text\":\"hello back\"}";

        var loop = new ChatAgencyLoop(llm);
        loop.setSystemPrompt("You are a friendly bot.");

        var context = new AgencyContext(new NeedState());
        var msg = new ReceivedMessage("discord", new ChatChannelRef("ch-1"),
                new ChatMessageRef(new ChatChannelRef("ch-1"), "m1"), null,
                new MemberRef("user-1"),
                new ChatContent("hello", null, List.of(), List.of()), Instant.now());
        var perception = new ChatPerception(
                Map.of("ch-1", List.of(msg)), Map.of(), WakeReason.MESSAGE);
        context.put("perception", perception);

        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");
        assertNotNull(intents);
        assertFalse(intents.isEmpty());
    }

    @Test
    void heartbeatWithNoActivityProducesNoIntents() {
        var llm = (ChatAgencyLoop.LlmInvoker) (system, user, id) ->
                "{\"action\":\"WAIT\"}";

        var loop = new ChatAgencyLoop(llm);
        var context = new AgencyContext(new NeedState());
        var perception = new ChatPerception(Map.of(), Map.of(), WakeReason.HEARTBEAT);
        context.put("perception", perception);

        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");
        assertTrue(intents == null || intents.isEmpty());
    }
}
```

- [ ] **Step 4: Implement ChatAgencyLoop**

```java
package io.quarkmind.chat.agent;

import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.AgencyLoop;
import io.quarkmind.agency.chat.ChatDeltaReport;
import io.quarkmind.chat.protocol.*;
import java.util.*;

public class ChatAgencyLoop implements AgencyLoop {

    @FunctionalInterface
    public interface LlmInvoker {
        String invoke(String systemPrompt, String userPrompt, String agentId);
    }

    private final LlmInvoker llmInvoker;
    private String systemPrompt = "";
    private String agentId = "chat-agent";

    public ChatAgencyLoop(LlmInvoker llmInvoker) {
        this.llmInvoker = llmInvoker;
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

        String userPrompt = buildUserPrompt(perception, context);
        String response = llmInvoker.invoke(systemPrompt, userPrompt, agentId);
        List<ChatIntent> intents = parseResponse(response);
        context.put("intents", intents);
    }

    private String buildUserPrompt(ChatPerception perception, AgencyContext context) {
        var sb = new StringBuilder();
        sb.append("Needs: SOCIAL=%.0f, CURIOSITY=%.0f, EXPRESSION=%.0f\n".formatted(
                context.needState().get("SOCIAL"),
                context.needState().get("CURIOSITY"),
                context.needState().get("EXPRESSION")));

        for (var entry : perception.channelDeltas().entrySet()) {
            sb.append("\nChannel #").append(entry.getKey()).append(":\n");
            for (var msg : entry.getValue()) {
                sb.append("  ").append(msg.sender().id()).append(": ")
                        .append(msg.content().text()).append("\n");
            }
        }

        sb.append("""

                Respond with JSON:
                {"action":"SEND|REPLY|REACT|WAIT","channel":"channel-id","text":"message","emoji":"emoji","replyTo":"message-id"}
                Only include fields relevant to the action.
                """);
        return sb.toString();
    }

    static List<ChatIntent> parseResponse(String response) {
        var intents = new ArrayList<ChatIntent>();
        try {
            String action = extractField(response, "action");
            if (action == null || "WAIT".equalsIgnoreCase(action)) return intents;

            switch (action.toUpperCase()) {
                case "SEND" -> {
                    String channel = extractField(response, "channel");
                    String text = extractField(response, "text");
                    if (channel != null && text != null) {
                        intents.add(new ChatIntent.Send(channel,
                                new ChatContent(text, null, List.of(), List.of())));
                    }
                }
                case "REPLY" -> {
                    String replyTo = extractField(response, "replyTo");
                    String text = extractField(response, "text");
                    if (replyTo != null && text != null) {
                        var parentRef = new ChatMessageRef(new ChatChannelRef(""), replyTo);
                        intents.add(new ChatIntent.Reply(parentRef,
                                new ChatContent(text, null, List.of(), List.of())));
                    }
                }
                case "REACT" -> {
                    String msgId = extractField(response, "replyTo");
                    String emoji = extractField(response, "emoji");
                    if (msgId != null && emoji != null) {
                        var msgRef = new ChatMessageRef(new ChatChannelRef(""), msgId);
                        intents.add(new ChatIntent.React(msgRef, emoji));
                    }
                }
                default -> {}
            }
        } catch (Exception e) {
            // LLM response parsing failures are expected
        }
        return intents;
    }

    private static String extractField(String json, String field) {
        String pattern = "\"" + field + "\"";
        int idx = json.indexOf(pattern);
        if (idx < 0) return null;
        int colon = json.indexOf(':', idx + pattern.length());
        if (colon < 0) return null;
        int start = colon + 1;
        while (start < json.length() && json.charAt(start) == ' ') start++;
        if (start >= json.length()) return null;
        if (json.charAt(start) == '"') {
            int end = json.indexOf('"', start + 1);
            return end > start ? json.substring(start + 1, end) : null;
        }
        int end = start;
        while (end < json.length() && json.charAt(end) != ',' && json.charAt(end) != '}') end++;
        return json.substring(start, end).strip();
    }
}
```

- [ ] **Step 5: Run all tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): ChatWorldBridge + ChatAgencyLoop — chat agent skeleton Refs #279"
```

### Task 7: End-to-end integration test with ref ChatPlatform

**Files:**
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatAgentEndToEndTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1–6
- Produces: Verified end-to-end cycle: perception → agency loop → intent dispatch

- [ ] **Step 1: Write end-to-end test**

Test the full cycle: messages arrive → ChatWorldBridge builds perception → ChatAgencyLoop processes → intents dispatched to mock messaging:

```java
package io.quarkmind.chat.agent;

import io.casehub.connectors.chat.model.*;
import io.quarkmind.agency.AgencyContext;
import io.quarkmind.agency.chat.BotIdentityDetector;
import io.quarkmind.agency.needs.NeedState;
import io.quarkmind.agency.schedule.OutputGovernor;
import io.quarkmind.chat.protocol.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.atomic.AtomicReference;
import static org.junit.jupiter.api.Assertions.*;

class ChatAgentEndToEndTest {

    @Test
    void fullCycleFromMessageToResponse() {
        // Arrange: mock LLM returns a SEND intent
        var llm = (ChatAgencyLoop.LlmInvoker) (sys, usr, id) ->
                "{\"action\":\"SEND\",\"channel\":\"general\",\"text\":\"Hey there!\"}";

        var loop = new ChatAgencyLoop(llm);
        loop.setSystemPrompt("You are Quark, a friendly chat character.");

        // Mock message history
        var incomingMsg = new ReceivedMessage("discord", new ChatChannelRef("general"),
                new ChatMessageRef(new ChatChannelRef("general"), "m1"), null,
                new MemberRef("alice"),
                new ChatContent("hi everyone!", null, List.of(), List.of()), Instant.now());

        var history = new StubMessageHistory(Map.of("general", List.of(incomingMsg)));

        var detector = stubDetector();
        var bridge = new ChatWorldBridge(history, List.of("general"), detector, WakeReason.MESSAGE);

        // Track what gets sent
        var sent = new AtomicReference<String>();
        bridge.setMessaging((channel, content) -> {
            sent.set(content.text());
            return SendResult.success(new ChatMessageRef(channel, "sent-1"), Instant.now());
        });
        bridge.setThreading((parent, content) ->
                SendResult.success(new ChatMessageRef(parent.channel(), "sent-2"), Instant.now()));
        bridge.setReactions(new io.casehub.connectors.chat.degraded.NoOpReactions());

        // Act
        var perception = bridge.perceive();
        var context = new AgencyContext(new NeedState());
        context.put("perception", perception);
        loop.tick(context);

        @SuppressWarnings("unchecked")
        var intents = (List<ChatIntent>) context.get("intents");

        var queue = new io.quarkmind.agency.intent.IntentQueue<ChatIntent>();
        var governor = new OutputGovernor(300_000, 0, 10);
        for (ChatIntent intent : intents) {
            if (governor.allow()) {
                queue.add(intent);
                governor.recordAction();
            }
        }
        bridge.dispatch(queue);

        // Assert
        assertEquals("Hey there!", sent.get());
    }

    private BotIdentityDetector stubDetector() {
        return new BotIdentityDetector() {
            @Override public boolean isMention(ReceivedMessage msg) { return false; }
            @Override public boolean isReplyToBot(ReceivedMessage msg) { return false; }
            @Override public String botUserId() { return "bot-id"; }
        };
    }

    private record StubMessageHistory(Map<String, List<ReceivedMessage>> data)
            implements io.casehub.connectors.chat.spi.MessageHistory {
        @Override
        public List<ReceivedMessage> messages(ChatChannelRef channel, Instant since) {
            return data.getOrDefault(channel.id(), List.of());
        }
    }
}
```

- [ ] **Step 2: Run the test**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -Dtest=ChatAgentEndToEndTest -q`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add quarkmind-chat/quarkmind-chat-agent/
git commit -m "feat(#279): end-to-end integration test — full perception→reasoning→dispatch cycle Refs #279"
```

## Batch 4: Need system + configuration

### Task 8: ChatNeedDefinitions + config-driven needs

**Files:**
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/java/io/quarkmind/chat/agent/ChatNeedDefinitions.java`
- Create: `quarkmind-chat/quarkmind-chat-agent/src/main/resources/application.yaml`
- Test: `quarkmind-chat/quarkmind-chat-agent/src/test/java/io/quarkmind/chat/agent/ChatNeedDefinitionsTest.java`

**Interfaces:**
- Consumes: `NeedState`, `NeedDefinition`, `DispositionNeedModifier` from quarkmind-core
- Produces: `ChatNeedDefinitions` — SOCIAL, CURIOSITY, EXPRESSION with configurable decay rates and thresholds

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
        // Expression builds (inverse decay — increases over time)
        defs.buildExpression(needs);
        assertTrue(needs.get("EXPRESSION") > 0.0);
    }

    @Test
    void needsCrossThreshold() {
        var needs = new NeedState();
        needs.set("SOCIAL", 25.0);
        var defs = new ChatNeedDefinitions(0.5, 0.1, 0.2);
        assertTrue(defs.socialBelowThreshold(needs, 30.0));
        assertFalse(defs.socialBelowThreshold(needs, 20.0));
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

    public boolean socialBelowThreshold(NeedState needs, double threshold) {
        return needs.get("SOCIAL") < threshold;
    }

    public boolean curiosityAboveThreshold(NeedState needs, double threshold) {
        return needs.get("CURIOSITY") > threshold;
    }

    public boolean expressionAboveThreshold(NeedState needs, double threshold) {
        return needs.get("EXPRESSION") > threshold;
    }
}
```

- [ ] **Step 3: Create application.yaml with default config**

```yaml
# quarkmind-chat default configuration
quarkmind:
  chat:
    channels: general
    needs:
      social:
        decay-rate: 0.5
        satisfaction-threshold: 30
      curiosity:
        decay-rate: 0.1
        spike-threshold: 70
      expression:
        build-rate: 0.2
        satisfaction-threshold: 60
    governor:
      max-unprompted-per-window: 1
      window-seconds: 300
      min-interval-seconds: 30
    heartbeat-seconds: 60

# Discord profile
"%discord":
  casehub:
    discord:
      token: ${DISCORD_BOT_TOKEN:}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl quarkmind-chat/quarkmind-chat-agent -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add quarkmind-chat/
git commit -m "feat(#279): ChatNeedDefinitions + config-driven need system Refs #279"
```

---

## References

- `specs/issue-279-quarkmind-discord/2026-08-17-quarkmind-discord-design.md` — design spec this plan implements
- `specs/issue-279-quarkmind-discord/decisions.md` — D1–D12 design decisions
- `quarkmind-core/src/main/java/io/quarkmind/agency/` — existing agency framework SPIs
- `quarkmind-ville/quarkmind-ville-agent/src/main/java/io/quarkmind/ville/agent/VilleWorldBridge.java` — reference WorldBridge implementation
- `quarkmind-ville/quarkmind-ville-agent/src/main/java/io/quarkmind/ville/agent/VilleAgencyLoop.java` — reference AgencyLoop implementation
- `casehub-connectors/chat-spi/` — ChatPlatform SPI, MessageHistory, Messaging, Threading, Reactions
- `casehub-connectors/chat-discord/` — DiscordChatPlatform, DiscordInboundConnector
- `casehub-blocks/blocks/` — TieredObservationRenderer, EventSource, ChoreographedDriver
- GitHub #279 — quarkmind-discord — persistent character in Discord
