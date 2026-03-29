# Campus AI Agent - Architecture Design

## Overview

This document describes the architecture for an AI agent that helps students, faculty and staff on a college campus. The agent handles four main things:

- Answering questions about upcoming events (seminars, fests, workshops etc.)
- Checking room and lab availability
- Answering general facility queries (timingecho "# Campus AI Agent Project" >> README.md
git add README.md ARCHITECTURE.md
git commit -m "Initial commit: Architecture and README"
git push -u origin main
s, locations, rules)
- Handling room/lab bookings with proper confirmation

---

## High-Level Architecture

The system has three main layers:

```
User (chat interface)
        |
        v
   [FastAPI Backend]  -  handles auth, sessions, rate limiting
        |
        v
   [LangChain Agent / Orchestrator]  -  the brain of the system
        |
        +--> Intent Detection (what does the user want?)
        +--> Tool Selection (which system to query?)
        +--> Tool Execution (fetch data / perform action)
        +--> Response Generation (convert raw data to natural reply)
        |
        v
   [Data Layer]
        +--> Events Database (PostgreSQL)
        +--> Room/Lab Schedule Database (PostgreSQL)
        +--> Facility Records (PostgreSQL)
```

The user sends a message through a web chat or mobile app. FastAPI receives it, passes it to the LangChain agent, the agent figures out what to do, calls the appropriate tool, and sends back a human-friendly response.

---

## Internal Components

### 1. Intent Detection

The LLM reads the user's message and figures out which category it falls into:

- **EVENT_QUERY** - "what events are happening this Friday?", "any workshops this week?"
- **ROOM_LAB_AVAILABILITY** - "is room 301 free at 3pm?", "lab schedule for tomorrow"
- **FACILITY_INFO** - "library timings", "where is the seminar hall?"
- **BOOKING_REQUEST** - "book room 301 for Monday 2pm", "register for the hackathon"

This is done through the LLM's system prompt with clear definitions and a few examples. We don't need a separate classifier model for this - the LLM handles it well enough as part of its reasoning.

### 2. Router (Decision Point)

Once the intent is known, the agent picks which tool to call:

- Event query --> events_tool
- Room/lab availability --> room_lab_tool
- Facility info --> facility_tool
- Booking request --> room_lab_tool first (to check availability), then constraint_checker, then confirmation, then booking_tool

### 3. Tools

These are simple Python functions that talk to the database:

- **events_tool** - queries the events table by date, category, or keyword
- **room_lab_tool** - checks the room/lab schedule, returns free slots
- **facility_tool** - looks up facility details (location, hours, contact info)
- **constraint_checker** - validates booking rules before allowing a booking
- **booking_tool** - writes a new booking record to the database (only runs after user confirms)

### 4. Constraint Checker

Before any booking goes through, we check:

- Is the requested time within operating hours?
- Is the duration within the allowed limit (e.g. max 3 hours)?
- Does the user have permission? (students can't book certain admin rooms)
- Is there a maintenance block or holiday on that date?

If something fails, the agent explains why and suggests an alternative.

### 5. Confirmation Gate

This is important - the agent never books anything without asking the user first. The flow looks like:

```
Agent: "Physics Lab 2 is available tomorrow 3-5 PM. Should I go ahead and book it?"
User: "yes"
Agent: [executes booking] "Done! Booking confirmed. Ref: #BK-4821"
```

This isn't just the LLM being polite - there's a hard-coded flag (user_confirmed) that the booking_tool checks before executing. The tool literally won't run without it. This prevents accidental bookings.

### 6. Response Generator

The LLM takes raw database output (JSON) and turns it into a natural response. For example, `{"room": "301", "status": "available", "slot": "14:00-16:00"}` becomes "Room 301 is free from 2 to 4 PM. Want me to book it?"

---

## End-to-End Flow (Booking Example)

Let's walk through what happens when a user says "Book Physics Lab 2 for tomorrow 3-5 PM":

**Step 1 - Intent Detection**
The LLM reads the message and identifies it as a BOOKING_REQUEST. It also extracts: room = "Physics Lab 2", date = tomorrow, time = 3-5 PM.

**Step 2 - Availability Check**
The agent calls room_lab_tool, which queries the schedule database. Result: the lab is free during that slot.

**Step 3 - Constraint Validation**
The constraint_checker runs:

- Operating hours? Yes, lab is open till 6 PM.
- Duration ok? Yes, 2 hours is under the 3-hour limit.
- User has permission? Yes, students can book labs.
- Any maintenance? No.
  All checks pass.

**Step 4 - User Confirmation**
Agent asks: "Physics Lab 2 is available tomorrow 3-5 PM. Confirm booking?"
User replies: "Yes"
The confirmed flag is set to true.

**Step 5 - Execute Booking**
booking_tool writes the record to the database and returns a reference number.

**Step 6 - Response**
Agent replies: "Booked! Physics Lab 2, tomorrow 3-5 PM. Reference: #BK-4821."

---

## Tech Stack

| Component       | Technology                         | Why                                                                                             |
| --------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| LLM             | OpenAI GPT-4o                      | Good at reasoning and tool calling out of the box, supports function calling natively           |
| Agent Framework | LangChain (Python)                 | Gives us a ready-made agent loop with tool support and conversation memory, saves a lot of time |
| Backend         | FastAPI                            | Fast, async, auto-generates API docs, works well for chat APIs                                  |
| Database        | PostgreSQL                         | Solid relational DB, good for structured data like events and schedules                         |
| ORM             | SQLAlchemy                         | Clean Python-to-SQL mapping, integrates well with FastAPI                                       |
| Date Parsing    | python-dateutil                    | Handles "tomorrow", "next Monday" etc. and converts to actual dates                             |
| Memory          | LangChain ConversationBufferMemory | Keeps last few messages in context so the agent can handle follow-ups like "book that one"      |
| Deployment      | Docker + docker-compose            | One command to spin up the whole stack                                                          |

We don't need a separate NLP model for entity extraction - GPT-4o handles extracting room names, dates and times from natural language as part of its normal processing. We also don't need a vector database since all our data is structured (tables and schedules), not unstructured text.

---

## Flow Diagram

```
        User Query
            |
            v
     [Intent Detection]
      /      |       \
     v       v        v
  Events   Room/Lab  Facility
  Tool      Tool      Tool
     \       |        /
      \      v       /
       \  Booking?  /
        \  / No \  /
         v       v
     [Constraint Check]   (only for bookings)
         |          \
        Pass        Fail --> suggest alternative
         |
         v
     [Ask User to Confirm]
        / \
      No   Yes
      |     |
      v     v
    Cancel  [Execute Booking]
              |
              v
     [Generate Response]
              |
              v
          User gets reply
```

---

## Key Design Decisions

- **Single LLM for everything** - Instead of having separate models for intent classification, entity extraction etc., we use one LLM (GPT-4o) with a good system prompt. Simpler to maintain and works well for this scope.

- **Hard-coded confirmation gate** - The booking tool physically cannot run without the confirmed flag being true. This is a programmatic check, not just the LLM being careful. Safety first.

- **Tools are thin wrappers** - Each tool is just a function that builds a SQL query, runs it, and returns the result as JSON. Easy to test, easy to replace.

- **No vector database** - Our data is all structured (event tables, room schedules). There's no need for semantic search here, so adding a vector DB would just be unnecessary complexity.

- **No fine-tuning** - The domain is small enough that prompt engineering with a few examples is enough. Fine-tuning would be overkill for a campus assistant.
