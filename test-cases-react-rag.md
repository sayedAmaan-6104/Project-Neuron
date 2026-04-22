# Test Cases: ReAct Reasoning Loop & Multi-Stage RAG

> Covers `functions/src/services/chat-retrieval.ts` (backend) and
> `app/home/semantic-chat.tsx` (frontend) after the §1 + §2 implementation.

---

## 1. Backend — `semanticChat` Cloud Function

### 1.1 Auth & Input Validation

| #     | Case                          | Input                      | Expected                                   |
| ----- | ----------------------------- | -------------------------- | ------------------------------------------ |
| 1.1.1 | Unauthenticated request       | `request.auth = undefined` | `HttpsError('unauthenticated')`            |
| 1.1.2 | Empty query                   | `{ query: '' }`            | `HttpsError('invalid-argument')`           |
| 1.1.3 | Null query                    | `{ query: null }`          | `HttpsError('invalid-argument')`           |
| 1.1.4 | Non-string query              | `{ query: 42 }`            | `HttpsError('invalid-argument')`           |
| 1.1.5 | Whitespace-only query         | `{ query: '   ' }`         | `HttpsError('invalid-argument')`           |
| 1.1.6 | Oversized query (>2000 chars) | 3000-char string           | Truncated to 2000 chars, proceeds normally |

### 1.2 Query Expansion (`expandQuery`)

| #     | Case                         | Input                            | Expected                                                 |
| ----- | ---------------------------- | -------------------------------- | -------------------------------------------------------- |
| 1.2.1 | Normal query                 | `"meeting with Alice last week"` | Returns 1–3 string array of semantically diverse queries |
| 1.2.2 | Short/vague query            | `"stuff"`                        | Still returns ≥1 expanded query (fallback to original)   |
| 1.2.3 | LLM returns empty array      | Mocked empty response            | Falls back to `[originalQuery]`                          |
| 1.2.4 | LLM returns malformed output | Non-array output                 | Falls back to `[originalQuery]`                          |

### 1.3 Multi-Query Retrieval (`multiQueryRetrieve`)

| #     | Case                              | Input                                   | Expected                                                |
| ----- | --------------------------------- | --------------------------------------- | ------------------------------------------------------- |
| 1.3.1 | Results from multiple collections | 3 queries × 4 collections               | Deduplicates by composite key, returns `TaggedResult[]` |
| 1.3.2 | Duplicate docs across queries     | Same doc returned by 2 expanded queries | Appears only once in output                             |
| 1.3.3 | Empty results for one collection  | `keep_notes` returns 0 results          | Other collections' results returned normally            |
| 1.3.4 | Single collection throws error    | `docs_content` retrieval fails          | Warning logged, other 3 collections still searched      |
| 1.3.5 | All collections empty             | No vectors match threshold              | Returns `[]`                                            |
| 1.3.6 | Results with empty text           | Doc has `embeddingText: ''`             | Filtered out (not included in output)                   |

### 1.4 LLM Reranking (`rerankResults`)

| #     | Case                             | Input                                  | Expected                                           |
| ----- | -------------------------------- | -------------------------------------- | -------------------------------------------------- |
| 1.4.1 | ≤5 candidates                    | 3 `TaggedResult`s                      | Skips reranking, returns all candidates as-is      |
| 1.4.2 | >5 candidates with mixed scores  | 10 candidates, scores 2–9              | Returns top 5 with score ≥ 6, sorted by score desc |
| 1.4.3 | All candidates score < threshold | 8 candidates, all score 3              | Returns empty array                                |
| 1.4.4 | LLM returns empty rerank         | Mocked empty output                    | Falls back to first 5 candidates                   |
| 1.4.5 | Out-of-bounds index in rerank    | LLM returns `{ index: 99, score: 10 }` | Filtered out (index ≥ candidates.length)           |

### 1.5 Full `multiStageRAG` Pipeline

| #     | Case             | Input                              | Expected                                                                     |
| ----- | ---------------- | ---------------------------------- | ---------------------------------------------------------------------------- |
| 1.5.1 | Happy path       | Valid query + uid with synced data | Returns `{ context: string, sources: string[] }` with numbered source blocks |
| 1.5.2 | No relevant data | Query about unsycned topic         | Returns `{ context: '', sources: [] }`                                       |
| 1.5.3 | Large result set | 20+ raw candidates                 | Reranked down to ≤5                                                          |

### 1.6 ReAct Agent (Tool Calling)

| #     | Case                 | Input                                                         | Expected                                                               |
| ----- | -------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 1.6.1 | Simple RAG query     | `"What did Alice email me about?"`                            | Agent calls `searchWorkspace`, returns answer with sources             |
| 1.6.2 | Calendar query       | `"What meetings do I have tomorrow?"`                         | Agent calls `getCalendarEvents` with tomorrow's date                   |
| 1.6.3 | Task query           | `"What are my pending tasks?"`                                | Agent calls `getTasksByStatus` with `needsAction`                      |
| 1.6.4 | Specific email query | `"Find the email about the Q4 budget"`                        | Agent calls `getEmailDetail` with subject keywords                     |
| 1.6.5 | Multi-tool query     | `"Summarize my emails and check if I have meetings tomorrow"` | Agent calls ≥2 tools across turns                                      |
| 1.6.6 | No tools needed      | `"Hello, how are you?"`                                       | Agent responds without tool calls                                      |
| 1.6.7 | Max turns reached    | Query requiring >5 tool calls                                 | Agent returns best answer after 5 turns (does not loop forever)        |
| 1.6.8 | Agent timeout        | Processing takes >45s                                         | `Promise.race` rejects with timeout, `HttpsError('internal')` returned |

### 1.7 Tool: `searchWorkspace`

| #     | Case               | Input                           | Expected                                                                   |
| ----- | ------------------ | ------------------------------- | -------------------------------------------------------------------------- |
| 1.7.1 | Valid search       | `{ query: "project deadline" }` | Returns context string with numbered sources, appends to `_requestSources` |
| 1.7.2 | No auth in context | `context.auth.uid = undefined`  | Returns `"Error: no authenticated user."`                                  |
| 1.7.3 | No results found   | Query with no vector matches    | Returns `"No relevant results found for this query."`                      |

### 1.8 Tool: `getCalendarEvents`

| #     | Case                    | Input                                        | Expected                                                       |
| ----- | ----------------------- | -------------------------------------------- | -------------------------------------------------------------- |
| 1.8.1 | Events found            | `{ date: "2025-01-15" }` with 3 events       | Returns bullet list: `• Event — time (attendees) [video call]` |
| 1.8.2 | No events               | `{ date: "2025-12-31" }` empty day           | Returns `"No events found for 2025-12-31."`                    |
| 1.8.3 | All-day event           | Event with `start.date` only (no `dateTime`) | Shows `"All day"` for time                                     |
| 1.8.4 | Event with hangout link | Event with `hangoutLink` set                 | Includes `[video call]` tag                                    |
| 1.8.5 | No auth                 | Missing uid                                  | Returns error string                                           |

### 1.9 Tool: `getTasksByStatus`

| #     | Case                  | Input                                    | Expected                                                             |
| ----- | --------------------- | ---------------------------------------- | -------------------------------------------------------------------- |
| 1.9.1 | Pending tasks         | `{ status: "needsAction" }` with 5 tasks | Returns bullet list with due dates                                   |
| 1.9.2 | Completed tasks       | `{ status: "completed" }`                | Returns completed tasks list                                         |
| 1.9.3 | No tasks              | Empty collection                         | Returns `"No pending tasks found."` or `"No completed tasks found."` |
| 1.9.4 | Task without due date | Task with `due: undefined`               | Omits due date suffix                                                |
| 1.9.5 | >15 tasks             | 20 tasks in collection                   | Returns max 15 (Firestore limit)                                     |

### 1.10 Tool: `getEmailDetail`

| #      | Case                 | Input                                     | Expected                                   |
| ------ | -------------------- | ----------------------------------------- | ------------------------------------------ |
| 1.10.1 | Single keyword match | `{ subjectKeywords: "budget" }`           | Returns email with subject/from/date/body  |
| 1.10.2 | Multi-keyword match  | `{ subjectKeywords: "Q4 budget review" }` | All keywords must appear in subject        |
| 1.10.3 | No matches           | Keywords not in any subject               | Returns `'No emails found matching "..."'` |
| 1.10.4 | >3 matches           | 5 emails match keywords                   | Returns only first 3                       |
| 1.10.5 | Long email body      | Body >500 chars                           | Truncated to 500 chars                     |

### 1.11 Reasoning Trace Extraction (`extractReasoningTrace`)

| #      | Case                         | Input                                                 | Expected                                         |
| ------ | ---------------------------- | ----------------------------------------------------- | ------------------------------------------------ |
| 1.11.1 | Full ReAct cycle             | Messages with model text + toolRequest + toolResponse | Returns `[thought, action, observation]` steps   |
| 1.11.2 | No tool calls                | Single model text response                            | Returns `[{ type: 'thought', text: '...' }]`     |
| 1.11.3 | System/user messages         | Mixed roles in history                                | Skips system + user, only processes model + tool |
| 1.11.4 | Long observation             | Tool response >600 chars                              | Truncated to 600 chars with `…`                  |
| 1.11.5 | Multiple tool calls per turn | Model requests 2 tools in one message                 | Separate action step for each toolRequest        |

### 1.12 Response Format

| #      | Case                      | Expected response shape                                            |
| ------ | ------------------------- | ------------------------------------------------------------------ |
| 1.12.1 | Successful query          | `{ steps: ReActStep[], answer: string, sources: string[] }`        |
| 1.12.2 | Steps include finalAnswer | Last step has `type: 'finalAnswer'`, `text` matches `answer` field |
| 1.12.3 | Sources deduplicated      | No duplicate strings in `sources` array                            |

### 1.13 Source Citations

| #      | Case                      | Collection       | Expected citation                             |
| ------ | ------------------------- | ---------------- | --------------------------------------------- |
| 1.13.1 | Gmail with subject + from | `gmail_messages` | `"Email: Weekly standup — from alice@co.com"` |
| 1.13.2 | Gmail without from        | `gmail_messages` | `"Email: Weekly standup"`                     |
| 1.13.3 | Document with title       | `docs_content`   | `"Document: Q4 Report"`                       |
| 1.13.4 | Note without title        | `keep_notes`     | `"Note: Untitled note"`                       |
| 1.13.5 | Unknown collection        | `some_other`     | `"Source"`                                    |

---

## 2. Frontend — `semantic-chat.tsx`

### 2.1 Response Parsing

| #     | Case                     | API Response                                      | Expected UI State                                          |
| ----- | ------------------------ | ------------------------------------------------- | ---------------------------------------------------------- |
| 2.1.1 | Full response with steps | `{ steps: [...], answer: "...", sources: [...] }` | `ChatMessage` has `text`, `sources`, and `steps` populated |
| 2.1.2 | Response without steps   | `{ answer: "...", sources: [] }` (edge case)      | `steps` is `undefined`, no reasoning trace shown           |
| 2.1.3 | Empty sources            | `{ steps: [...], answer: "...", sources: [] }`    | No sources section rendered                                |

### 2.2 ReasoningTrace Component

| #     | Case                         | Input                                                                          | Expected                                                           |
| ----- | ---------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| 2.2.1 | Multiple trace steps         | 5 steps (2 thoughts, 2 actions, 1 observation)                                 | Shows collapsed toggle: "▸ Reasoning (5 steps)"                    |
| 2.2.2 | Toggle expand                | User taps toggle                                                               | Shows all steps with icons, changes to "▾ Reasoning (5 steps)"     |
| 2.2.3 | Toggle collapse              | User taps expanded toggle                                                      | Hides step details, returns to "▸"                                 |
| 2.2.4 | Only finalAnswer steps       | `steps: [{ type: 'finalAnswer', text: '...' }]`                                | `ReasoningTrace` returns null (nothing rendered)                   |
| 2.2.5 | ≤1 total steps               | Single finalAnswer step in message                                             | Reasoning trace section not rendered (`steps.length > 1` guard)    |
| 2.2.6 | Thought step rendering       | `{ type: 'thought', text: 'I need to search...' }`                             | Shows `💭 THOUGHT` header + italic text                            |
| 2.2.7 | Action step with tool        | `{ type: 'action', text: 'Calling searchWorkspace', tool: 'searchWorkspace' }` | Shows `🔧 ACTION — searchWorkspace`                                |
| 2.2.8 | Observation step             | Long observation text                                                          | Shows `📋 OBSERVATION`, truncated to 4 lines (`numberOfLines={4}`) |
| 2.2.9 | Thought step (no line limit) | Long thought text                                                              | Shows full text (no `numberOfLines` constraint)                    |

### 2.3 Message Bubble Layout

| #     | Case                            | Expected                                                        |
| ----- | ------------------------------- | --------------------------------------------------------------- |
| 2.3.1 | AI message with steps + sources | Reasoning trace → answer text → sources section (top to bottom) |
| 2.3.2 | AI message without steps        | Answer text → sources section (no trace section)                |
| 2.3.3 | User message                    | Right-aligned primary-colored bubble, no trace or sources       |
| 2.3.4 | Default trace state             | Reasoning trace starts collapsed                                |

### 2.4 Error Handling

| #     | Case                  | Trigger                | Expected                                                            |
| ----- | --------------------- | ---------------------- | ------------------------------------------------------------------- |
| 2.4.1 | 401 response          | Expired auth token     | "Your session has expired. Please sign in again."                   |
| 2.4.2 | 400 response          | Invalid argument       | "Your query was invalid. Please try rephrasing."                    |
| 2.4.3 | 504 / timeout message | Agent timeout          | "The request took too long. Please try a shorter or simpler query." |
| 2.4.4 | Network failure       | `fetch` throws         | Generic error message in AI bubble                                  |
| 2.4.5 | Malformed JSON        | Non-JSON response body | Error message in AI bubble (catch block)                            |

### 2.5 Loading State

| #     | Case           | Expected                                             |
| ----- | -------------- | ---------------------------------------------------- |
| 2.5.1 | During request | Loading spinner + "Synthesizing response..." shown   |
| 2.5.2 | After response | Loading indicator removed                            |
| 2.5.3 | During loading | Send button disabled                                 |
| 2.5.4 | After error    | Loading indicator removed, error shown as AI message |

### 2.6 Styling Consistency

| #     | Case             | Expected                                                                      |
| ----- | ---------------- | ----------------------------------------------------------------------------- |
| 2.6.1 | Trace container  | Bottom border uses `outlineVariant + '33'` (consistent with other separators) |
| 2.6.2 | Toggle text      | Uses `theme.typography.styles.labelMD` + `onSurfaceVariant` color             |
| 2.6.3 | Step header      | `theme.colors.primary` for emphasis                                           |
| 2.6.4 | Step text        | `theme.colors.onSurfaceVariant`, italic                                       |
| 2.6.5 | Step left border | `outlineVariant + '55'` for subtle visual hierarchy                           |

---

## 3. Integration Tests

### 3.1 End-to-End Flow

| #     | Case                     | Steps                                                    | Expected                                                                     |
| ----- | ------------------------ | -------------------------------------------------------- | ---------------------------------------------------------------------------- |
| 3.1.1 | Full happy path          | User sends query → backend runs ReAct → frontend renders | Answer displayed with collapsible reasoning trace and sources                |
| 3.1.2 | Multi-tool query         | User asks about emails AND calendar                      | Response includes multiple action/observation steps from different tools     |
| 3.1.3 | No data scenario         | New user with no synced data                             | Agent tools return "no results" observations, final answer suggests syncing  |
| 3.1.4 | Large context            | Query matches many documents across all 4 collections    | Reranking reduces to ≤5, answer is coherent and cited                        |
| 3.1.5 | Rapid sequential queries | User sends 2 queries back-to-back                        | `_requestSources` reset per invocation — sources don't leak between requests |

### 3.2 Source Accumulation

| #     | Case                  | Expected                                                             |
| ----- | --------------------- | -------------------------------------------------------------------- |
| 3.2.1 | Single tool call      | Sources from `searchWorkspace` appear in final `sources` array       |
| 3.2.2 | Multiple tool calls   | Sources from all `searchWorkspace` calls are merged and deduplicated |
| 3.2.3 | Non-search tool calls | `getCalendarEvents` / `getTasksByStatus` don't add to sources        |
| 3.2.4 | Request isolation     | Second request starts with empty `_requestSources`                   |
