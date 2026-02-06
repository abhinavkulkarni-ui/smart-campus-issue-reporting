# Design Document: Smart Campus Issue Reporting System

## Overview

The Smart Campus Issue Reporting System is a web-based application that facilitates efficient communication between students and campus administrators regarding infrastructure and facility issues. The system follows a three-tier architecture with a React-based frontend, a RESTful API backend built with Node.js/Express, and a PostgreSQL database for persistent storage.

The design emphasizes simplicity, reliability, and user experience. Students can quickly report issues through an intuitive interface, while administrators have powerful tools for managing and resolving reports through a comprehensive dashboard. The system maintains complete audit trails of all status changes and provides real-time notifications to keep users informed.

## Architecture

### System Architecture

The system follows a client-server architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer (React)                     │
│  ┌──────────────────┐         ┌──────────────────────────┐  │
│  │  Student Portal  │         │  Admin Dashboard         │  │
│  │  - Report Issues │         │  - View All Issues       │  │
│  │  - Track Status  │         │  - Update Status         │  │
│  │  - Notifications │         │  - Search & Filter       │  │
│  └──────────────────┘         └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/REST API
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Application Layer (Node.js/Express)             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Auth Service │  │Issue Service │  │Notification Svc  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ SQL
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Data Layer (PostgreSQL)                     │
│     Users │ Issues │ Status_History │ Notifications         │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Frontend**: React with TypeScript, React Router for navigation, Axios for API calls
- **Backend**: Node.js with Express framework, TypeScript
- **Database**: PostgreSQL for relational data storage
- **Authentication**: JWT (JSON Web Tokens) for stateless authentication
- **API Design**: RESTful principles with JSON payloads

## Components and Interfaces

### Frontend Components

#### Student Portal Components

1. **IssueReportForm**
   - Purpose: Capture new issue reports from students
   - Inputs: Category dropdown, Location fields (building, room/area), Description textarea
   - Validation: Required field checks, location format validation
   - Output: Submits issue data to backend API

2. **MyIssuesView**
   - Purpose: Display all issues reported by the logged-in student
   - Features: List view with status badges, click to view details
   - Real-time updates: Polls for status changes or uses WebSocket

3. **IssueDetailView**
   - Purpose: Show complete details of a single issue
   - Displays: All issue fields, status history, administrator comments
   - Actions: None (read-only for students)

4. **NotificationPanel**
   - Purpose: Display notifications about issue status updates
   - Features: Unread badge, mark as read functionality
   - Updates: Real-time notification delivery

#### Admin Dashboard Components

1. **IssuesDashboard**
   - Purpose: Central view of all reported issues
   - Features: Sortable table, filter controls, search bar
   - Displays: Issue ID, category, location, status, submission date, reporter

2. **IssueManagementView**
   - Purpose: Detailed view with status update controls
   - Features: Status dropdown, comment textarea, update button
   - Validation: Requires comment for Resolved/Closed status

3. **FilterPanel**
   - Purpose: Provide filtering and search capabilities
   - Controls: Category checkboxes, status checkboxes, location search, date range
   - Behavior: Applies filters using AND logic

### Backend Services

#### Authentication Service

```typescript
interface AuthService {
  // Authenticate user credentials
  login(email: string, password: string): Promise<AuthResult>
  
  // Validate JWT token and extract user info
  verifyToken(token: string): Promise<User>
  
  // Create new user account
  register(userData: UserRegistration): Promise<User>
}

interface AuthResult {
  token: string
  user: User
  expiresIn: number
}

interface User {
  id: string
  email: string
  name: string
  role: 'student' | 'administrator'
}
```

#### Issue Service

```typescript
interface IssueService {
  // Create a new issue report
  createIssue(issueData: IssueCreationData, reporterId: string): Promise<Issue>
  
  // Retrieve issues with optional filters
  getIssues(filters: IssueFilters, userId: string, userRole: string): Promise<Issue[]>
  
  // Get a single issue by ID
  getIssueById(issueId: string, userId: string, userRole: string): Promise<Issue>
  
  // Update issue status (admin only)
  updateIssueStatus(issueId: string, newStatus: IssueStatus, comment: string, adminId: string): Promise<Issue>
  
  // Search issues by text
  searchIssues(searchTerm: string, userRole: string): Promise<Issue[]>
}

interface IssueCreationData {
  category: Category
  location: Location
  description: string
}

interface IssueFilters {
  categories?: Category[]
  statuses?: IssueStatus[]
  locations?: string[]
  dateFrom?: Date
  dateTo?: Date
}
```

#### Notification Service

```typescript
interface NotificationService {
  // Create notification for status update
  createNotification(issueId: string, recipientId: string, message: string): Promise<Notification>
  
  // Get unread notifications for user
  getUnreadNotifications(userId: string): Promise<Notification[]>
  
  // Mark notification as read
  markAsRead(notificationId: string, userId: string): Promise<void>
  
  // Get all notifications for user
  getAllNotifications(userId: string): Promise<Notification[]>
}

interface Notification {
  id: string
  userId: string
  issueId: string
  message: string
  isRead: boolean
  createdAt: Date
}
```

### REST API Endpoints

#### Authentication Endpoints

- `POST /api/auth/login` - Authenticate user and return JWT
- `POST /api/auth/register` - Register new user account
- `GET /api/auth/me` - Get current user info from token

#### Issue Endpoints

- `POST /api/issues` - Create new issue report (student)
- `GET /api/issues` - Get issues (filtered by role: students see only their own)
- `GET /api/issues/:id` - Get specific issue details
- `PATCH /api/issues/:id/status` - Update issue status (admin only)
- `GET /api/issues/search?q=<term>` - Search issues by text

#### Notification Endpoints

- `GET /api/notifications` - Get all notifications for current user
- `GET /api/notifications/unread` - Get unread notifications
- `PATCH /api/notifications/:id/read` - Mark notification as read

## Data Models

### Database Schema

#### Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL CHECK (role IN ('student', 'administrator')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Issues Table

```sql
CREATE TABLE issues (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  reporter_id UUID NOT NULL REFERENCES users(id),
  category VARCHAR(50) NOT NULL CHECK (category IN ('Electrical', 'Plumbing', 'Cleanliness', 'Network', 'Other')),
  building VARCHAR(255) NOT NULL,
  room_area VARCHAR(255) NOT NULL,
  description TEXT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'Pending' CHECK (status IN ('Pending', 'In Progress', 'Resolved', 'Closed')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Status History Table

```sql
CREATE TABLE status_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  issue_id UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
  old_status VARCHAR(20),
  new_status VARCHAR(20) NOT NULL,
  comment TEXT,
  updated_by UUID NOT NULL REFERENCES users(id),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Notifications Table

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  issue_id UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
  message TEXT NOT NULL,
  is_read BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Buildings Reference Table

```sql
CREATE TABLE buildings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) UNIQUE NOT NULL,
  code VARCHAR(50) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### TypeScript Data Models

```typescript
enum Category {
  Electrical = 'Electrical',
  Plumbing = 'Plumbing',
  Cleanliness = 'Cleanliness',
  Network = 'Network',
  Other = 'Other'
}

enum IssueStatus {
  Pending = 'Pending',
  InProgress = 'In Progress',
  Resolved = 'Resolved',
  Closed = 'Closed'
}

interface Location {
  building: string
  roomArea: string
}

interface Issue {
  id: string
  reporterId: string
  reporterName: string
  category: Category
  location: Location
  description: string
  status: IssueStatus
  createdAt: Date
  updatedAt: Date
  statusHistory?: StatusHistoryEntry[]
}

interface StatusHistoryEntry {
  id: string
  issueId: string
  oldStatus: IssueStatus | null
  newStatus: IssueStatus
  comment: string | null
  updatedBy: string
  updatedByName: string
  updatedAt: Date
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Issue Creation Properties

**Property 1: Issue creation completeness**
*For any* valid issue data (category, location, description), creating an issue should return a unique identifier and persist an issue record containing all provided fields plus a Pending status and timestamp.
**Validates: Requirements 1.1, 1.2, 1.4, 1.5**

**Property 2: Invalid issue rejection**
*For any* issue data with missing required fields (category, building, room/area, or description), the system should reject the creation and return validation errors.
**Validates: Requirements 1.3**

### Issue Retrieval Properties

**Property 3: Reporter isolation**
*For any* student user, retrieving issues should return only the issues created by that specific reporter, and no issues from other reporters.
**Validates: Requirements 2.1**

**Property 4: Administrator visibility**
*For any* administrator user, retrieving issues should return all issues in the system regardless of which reporter created them.
**Validates: Requirements 3.1**

**Property 5: Issue serialization completeness**
*For any* issue retrieved from the system, the returned data should include all fields: unique identifier, category, location (building and room/area), description, current status, submission timestamp, and reporter information.
**Validates: Requirements 2.2, 3.4, 6.4, 7.4**

### Status Management Properties

**Property 6: Status update persistence**
*For any* issue and valid status change, updating the status should immediately reflect in subsequent retrievals of that issue.
**Validates: Requirements 2.3**

**Property 7: Status history tracking**
*For any* issue with status updates, the system should maintain a complete history of all status transitions including old status, new status, timestamp, and the administrator who made the change.
**Validates: Requirements 8.2**

**Property 8: Status comment visibility**
*For any* issue with status updates that include comments, retrieving the issue should include all administrator comments in the status history.
**Validates: Requirements 2.4**

**Property 9: Valid status transitions**
*For any* issue, only valid status transitions should be accepted (e.g., Pending → In Progress → Resolved → Closed), and invalid transitions should be rejected.
**Validates: Requirements 4.1**

**Property 10: Required comments for resolution**
*For any* status update to Resolved or Closed, the system should reject the update if no comment is provided, and accept it when a comment is included.
**Validates: Requirements 4.4**

### Category and Location Properties

**Property 11: Category validation**
*For any* issue creation, the system should accept only the predefined categories (Electrical, Plumbing, Cleanliness, Network, Other) and reject any other category values.
**Validates: Requirements 6.1, 6.2**

**Property 12: Other category description requirement**
*For any* issue with category "Other", the system should require a description of sufficient length (non-empty and meaningful), and reject issues with category "Other" that have insufficient descriptions.
**Validates: Requirements 6.3**

**Property 13: Location requirement**
*For any* issue creation attempt without location data (missing building or room/area), the system should reject the creation.
**Validates: Requirements 7.1**

**Property 14: Building validation**
*For any* issue creation, the system should validate that the specified building exists in the campus building registry, accepting valid buildings and rejecting invalid ones.
**Validates: Requirements 7.3**

**Property 15: Location structure preservation**
*For any* issue created with structured location data (building and room/area), retrieving the issue should return the location with the same structure and values.
**Validates: Requirements 7.2**

### Authentication Properties

**Property 16: Valid authentication success**
*For any* user with valid credentials (correct email and password), authentication should succeed and return a token and user information including role.
**Validates: Requirements 5.1, 5.4**

**Property 17: Invalid authentication rejection**
*For any* authentication attempt with invalid credentials (wrong email or wrong password), the system should reject the authentication and return an error.
**Validates: Requirements 5.2**

### Persistence Properties

**Property 18: Issue persistence**
*For any* successfully created issue, immediately querying for that issue by its ID should return the issue with all its data intact.
**Validates: Requirements 8.1**

### Notification Properties

**Property 19: Notification creation on status update**
*For any* status update performed by an administrator, the system should create a notification for the issue reporter containing the issue ID, new status, and any administrator comment.
**Validates: Requirements 9.1, 9.2**

**Property 20: Unread notification filtering**
*For any* user with both read and unread notifications, retrieving unread notifications should return only those marked as unread.
**Validates: Requirements 9.3**

**Property 21: Notification read state update**
*For any* unread notification, marking it as read should change its state so that subsequent retrievals show it as read.
**Validates: Requirements 9.4**

### Search and Filter Properties

**Property 22: Search matching**
*For any* search term, the system should return all issues where the description or location contains that term, and exclude issues that don't match.
**Validates: Requirements 10.1**

**Property 23: Filter combination**
*For any* set of filter criteria (category, status, location), the system should return only issues that match all specified criteria (logical AND operation).
**Validates: Requirements 3.2, 10.2, 10.3**

**Property 24: Timestamp sorting**
*For any* set of issues, sorting by submission timestamp should return them in chronological order (oldest to newest or newest to oldest as specified).
**Validates: Requirements 3.3**

## Error Handling

### Validation Errors

The system implements comprehensive input validation at multiple layers:

1. **Frontend Validation**: Immediate feedback for required fields, format validation
2. **API Validation**: Server-side validation before processing requests
3. **Database Constraints**: Enforce data integrity at the database level

**Error Response Format**:
```typescript
interface ErrorResponse {
  error: string
  message: string
  details?: ValidationError[]
}

interface ValidationError {
  field: string
  message: string
}
```

### Common Error Scenarios

1. **Missing Required Fields**: Return 400 Bad Request with field-specific error messages
2. **Invalid Category**: Return 400 Bad Request indicating valid category options
3. **Invalid Building**: Return 400 Bad Request with message about building not found
4. **Unauthorized Access**: Return 401 Unauthorized for missing/invalid tokens
5. **Forbidden Actions**: Return 403 Forbidden when students attempt admin actions
6. **Resource Not Found**: Return 404 Not Found for non-existent issues
7. **Invalid Status Transition**: Return 400 Bad Request with valid transition information
8. **Missing Required Comment**: Return 400 Bad Request when Resolved/Closed lacks comment

### Error Handling Strategy

- All errors are logged server-side with context for debugging
- User-facing error messages are clear and actionable
- Sensitive information (stack traces, internal details) is never exposed to clients
- Database errors are caught and translated to appropriate HTTP responses
- Transaction rollbacks ensure data consistency on errors

## Testing Strategy

### Dual Testing Approach

The system will employ both unit testing and property-based testing to ensure comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs using randomized test data

Both approaches are complementary and necessary. Unit tests catch concrete bugs and validate specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing Configuration

**Library Selection**: 
- **Backend (TypeScript/Node.js)**: Use `fast-check` library for property-based testing
- **Frontend (React/TypeScript)**: Use `fast-check` for business logic testing

**Test Configuration**:
- Each property test must run a minimum of 100 iterations to ensure adequate coverage
- Each test must include a comment tag referencing the design document property
- Tag format: `// Feature: smart-campus-issue-reporting, Property N: [property description]`

**Example Property Test Structure**:
```typescript
// Feature: smart-campus-issue-reporting, Property 1: Issue creation completeness
test('creating valid issue returns unique ID and persists all fields', async () => {
  await fc.assert(
    fc.asyncProperty(
      validIssueDataArbitrary(),
      async (issueData) => {
        const result = await issueService.createIssue(issueData, testUserId);
        
        expect(result.id).toBeDefined();
        expect(result.category).toBe(issueData.category);
        expect(result.location).toEqual(issueData.location);
        expect(result.description).toBe(issueData.description);
        expect(result.status).toBe('Pending');
        expect(result.createdAt).toBeInstanceOf(Date);
        
        // Verify persistence
        const retrieved = await issueService.getIssueById(result.id, testUserId, 'student');
        expect(retrieved).toEqual(result);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing Strategy

Unit tests should focus on:

1. **Specific Examples**: Concrete test cases that demonstrate correct behavior
2. **Edge Cases**: Empty results, boundary conditions, special characters
3. **Error Conditions**: Invalid inputs, unauthorized access, missing resources
4. **Integration Points**: API endpoint behavior, database interactions, authentication flow

**Example Unit Test**:
```typescript
describe('IssueService.createIssue', () => {
  it('should reject issue with missing description', async () => {
    const invalidIssue = {
      category: 'Electrical',
      location: { building: 'Main Hall', roomArea: 'Room 101' },
      description: '' // Invalid: empty description
    };
    
    await expect(
      issueService.createIssue(invalidIssue, testUserId)
    ).rejects.toThrow('Description is required');
  });
  
  it('should reject issue with invalid building', async () => {
    const invalidIssue = {
      category: 'Electrical',
      location: { building: 'NonExistentBuilding', roomArea: 'Room 101' },
      description: 'Light not working'
    };
    
    await expect(
      issueService.createIssue(invalidIssue, testUserId)
    ).rejects.toThrow('Building not found');
  });
});
```

### Test Coverage Goals

- **Backend Services**: 90%+ code coverage
- **API Endpoints**: All endpoints tested with success and error cases
- **Frontend Components**: Critical user flows and state management
- **Property Tests**: All 24 correctness properties implemented as property-based tests
- **Integration Tests**: End-to-end flows for issue creation, status updates, and notifications

### Testing Tools

- **Unit Testing**: Jest for both frontend and backend
- **Property Testing**: fast-check library
- **API Testing**: Supertest for HTTP endpoint testing
- **Database Testing**: In-memory PostgreSQL or test database with cleanup
- **Frontend Testing**: React Testing Library for component tests
