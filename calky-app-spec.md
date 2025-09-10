# Calky: iCalendar/Webcal on Pubky - Development Specification

## Project Overview

Calky is a decentralized calendar application built on Pubky Core, following iCalendar standards (RFC 5545, RFC 7986) for calendar data management. The application enables public calendar sharing while maintaining user ownership of data through Pubky's homeserver architecture. All calendars and events are publicly accessible by default, following Pubky's open data philosophy.

## Architecture Pattern

- **Frontend**: Next.js application that aggregates calendar data from multiple Pubky homeservers
- **Data Storage**: Each user stores their calendar configurations and events on their own homeserver
- **Discovery**: Uses Nexus patterns for discovering public calendars and events
- **Standards Compliance**: Full iCalendar (RFC 5545) compatibility with proper VEVENT structures
- **Public Data**: All calendar data is publicly accessible, with write permissions controlled by user authentication

## Data Structure

### Homeserver Paths

- **Base path**: `/pub/calky.app/`
- **Calendar configs**: `/pub/calky.app/calendars/{calendar-uuid}`
- **Events referencing calendars**: `/pub/calky.app/events/{calendar-uuid}/{event-uuid}`
- **User's own events**: `/pub/calky.app/events/{event-uuid}` (for events not tied to shared calendars)

### Calendar Configuration Structure

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "version": "1.0.0",
  "created_at": "2025-09-08T14:15:00Z",
  "updated_at": "2025-09-08T14:15:00Z",
  "owner": "pubky://admin-pubkey/",
  "metadata": {
    "name": "Team Engineering Meetings",
    "description": "Weekly engineering team coordination",
    "timezone": "America/New_York",
    "color": "#3b82f6"
  },
  "admins": [
    {
      "pubkey": "pubky://alice-pubkey/",
      "granted_at": "2025-09-08T14:15:00Z"
    },
    {
      "pubkey": "pubky://bob-pubkey/",
      "granted_at": "2025-09-08T14:16:00Z"
    }
  ],
  "settings": {
    "allow_external_events": true,
    "max_events_per_user": 100
  }
}
```

### Event Structure (RFC 5545 Compliant)

```json
{
  "id": "660f9500-f39c-41e4-b827-556766550000",
  "version": "1.0.0",
  "created_at": "2025-09-08T14:20:00Z",
  "updated_at": "2025-09-08T14:20:00Z",
  "author": "pubky://alice-pubkey/",
  "calendar_references": ["550e8400-e29b-41d4-a716-446655440000"],
  "icalendar": {
    "vcalendar": {
      "version": "2.0",
      "prodid": "-//Calky//Calky 1.0//EN",
      "vevent": {
        "uid": "660f9500-f39c-41e4-b827-556766550000@calky.app",
        "dtstamp": "20250908T141500Z",
        "dtstart": "20250910T140000Z",
        "dtend": "20250910T150000Z",
        "summary": "Weekly Engineering Standup",
        "description": "Review sprint progress and plan upcoming work",
        "location": "Conference Room A / Zoom",
        "organizer": "pk://alice-pubkey",
        "attendee": [
          "CN=Alice Smith:MAILTO:alice@company.com",
          "CN=Bob Johnson:MAILTO:bob@company.com"
        ],
        "categories": ["meeting", "engineering"],
        "status": "CONFIRMED",
        "transp": "OPAQUE",
        "sequence": 0
      }
    }
  }
}
```

### Tag Structure (Following Pubky App Pattern)

Tags are created as separate objects that reference calendars or events, following the exact same pattern as Pubky App:

```json
{
  "id": "tag-uuid-here",
  "uri": "pubky://tagger-pubkey/pub/calky.app/tags/{tag-id}",
  "created_at": "2025-09-08T14:25:00Z",
  "tagger": "pubky://alice-pubkey/",
  "label": "standup",
  "target": "pubky://alice-pubkey/pub/calky.app/events/660f9500-f39c-41e4-b827-556766550000"
}
```

## Application Features

### Core Functionality

1. **Calendar Overview Dashboard**

   - List all public calendars discovered through Nexus
   - Show calendar metadata and recent activity
   - Create new calendars on user's homeserver

2. **Event Management**

   - Create, edit, delete events within user's own calendars
   - Full iCalendar property support (recurrence, reminders, attendees)
   - Event tagging system following Pubky App patterns

3. **Calendar Views**

   - Month, week, day, and agenda views
   - Filter events by calendar, tags, or categories
   - Real-time aggregation from multiple homeservers

4. **Collaboration Features**
   - Calendar owners can designate admin users who are authorized to create events under the calendar UUID
   - Admins create events on their own homeservers at `/pub/calky.app/events/{calendar-uuid}/{event-uuid}`
   - Frontend aggregates all events referencing the same calendar UUID from all homeservers
   - Tag calendars and events from any user
   - Calendar subscription and following mechanisms

### Discovery Mechanism

Following Pubky.app patterns for public content discovery:

1. **Notification-Based Discovery**

   - Users receive notifications when designated as calendar admins
   - Notifications when calendars or events are tagged
   - Similar to Pubky's tag_post and mention notifications

2. **Bootstrap Integration**

   - User's owned calendars included in bootstrap payload
   - Recently tagged calendars and events in bootstrap
   - Cache calendar configurations for offline access

3. **Search and Indexing**
   - All calendars discoverable through Nexus search (public by default)
   - Tag-based calendar and event discovery (e.g., #team, #meeting)
   - Calendar and event streams similar to post streams

## Technical Implementation

### Frontend Components

```typescript
// Core calendar components
- CalendarDashboard: Overview of discoverable calendars
- CalendarView: Main calendar display (month/week/day views)
- EventForm: Create/edit event form with iCalendar validation
- AdminManager: Manage calendar admins (who can create events under calendar UUID)
- EventDetails: Full event information display
- TagManager: Tag calendars and events following Pubky patterns

// Data management
- CalendarAggregator: Fetches and merges events from multiple homeservers
- EventValidator: Ensures RFC 5545 compliance
- TagValidator: Manages tags following Pubky App patterns
```

### API Integration Patterns

#### Calendar Discovery (Following Pubky Nexus patterns)

```typescript
// GET /v0/bootstrap/{user_id}
// Returns user's calendars and recently tagged content
interface CalendarBootstrap {
  owned_calendars: CalendarConfig[];
  tagged_calendars: TaggedCalendar[];
  recent_events: Event[];
  notifications: CalendarNotification[];
}

// GET /v0/search/calendars/by_tag/{tag}
// Search calendars by tags
interface CalendarSearchResult {
  calendar_id: string;
  owner: string;
  metadata: CalendarMetadata;
  tag_count: number;
}

// GET /v0/stream/calendars
// Stream of public calendars (similar to post stream)
interface CalendarStream {
  calendars: CalendarView[];
  pagination: PaginationInfo;
}
```

#### Event Aggregation

```typescript
// GET /v0/stream/events
// Similar to Pubky's post stream, but for calendar events
interface EventStream {
  events: EventView[];
  pagination: PaginationInfo;
  calendar_sources: string[]; // Which homeservers provided data
}

// POST /v0/events (via homeserver)
// Create new event on user's homeserver
interface CreateEventRequest {
  calendar_id: string;
  icalendar_data: VEvent;
}

// GET /v0/events/{author_id}/{event_id}
// Get specific event details
interface EventView {
  details: EventDetails;
  tags: TagDetails[];
  calendar_ref: CalendarReference;
}
```

### Homeserver Integration

```typescript
// Direct homeserver operations (user's own data)
class CalkyHomeserverClient {
  async createCalendar(config: CalendarConfig): Promise<string>;
  async updateCalendarAdmins(
    calendarId: string,
    admins: Admin[]
  ): Promise<void>;
  async createEvent(event: Event, calendarId?: string): Promise<string>;
  async updateEvent(eventId: string, event: Partial<Event>): Promise<void>;
  async deleteEvent(eventId: string): Promise<void>;
  async createTag(tag: Tag): Promise<string>;
}

// Nexus integration for aggregation and discovery
class CalkyNexusClient {
  async getCalendarBootstrap(userId: string): Promise<CalendarBootstrap>;
  async searchCalendars(query: SearchQuery): Promise<CalendarSearchResult[]>;
  async getEventStream(filters: EventStreamFilters): Promise<EventStream>;
  async getCalendarNotifications(
    userId: string
  ): Promise<CalendarNotification[]>;
  async getCalendarTags(calendarId: string): Promise<TagDetails[]>;
  async getEventTags(eventId: string): Promise<TagDetails[]>;
}
```

### Access Control Flow

1. **Calendar Ownership**: Only calendar owners can modify calendar configuration and add/remove admins
2. **Event Creation**:
   - Calendar owners can always create events under their calendar UUID
   - Designated admins can create events on their own homeservers under the calendar UUID path
   - Events are stored at `/pub/calky.app/events/{calendar-uuid}/{event-uuid}` on each user's homeserver
3. **Event Modification**: Users can only modify events they created on their own homeserver
4. **Event Aggregation**: Frontend aggregates all events from all homeservers that reference the same calendar UUID
5. **Public Access**: All calendars and events are publicly readable by default
6. **Tagging**: Any user can tag any calendar or event (following Pubky App patterns)
7. **iCalendar Compliance**: Validate all event data follows RFC 5545 standards

### Error Handling

- **Access Denied**: Clear messaging when users try to modify calendars/events they don't own
- **Homeserver Unavailable**: Graceful degradation when source homeservers are offline
- **Data Conflicts**: Resolution strategies for conflicting event modifications
- **iCalendar Validation Errors**: User-friendly validation messages for invalid event data
- **Tag Validation**: Ensure tags follow Pubky App patterns and constraints

## Standards Compliance

### iCalendar (RFC 5545) Requirements

- Full VEVENT component support with all standard properties
- Proper timezone handling (VTIMEZONE components)
- Recurrence rule (RRULE) parsing and generation
- Free/busy time calculation support
- VALARM (reminder) component support

### Extended Properties (RFC 7986)

- COLOR property for calendar and event styling
- NAME and DESCRIPTION properties for calendar metadata
- Enhanced CATEGORIES support for event organization
