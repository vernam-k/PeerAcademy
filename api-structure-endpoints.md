# Peer Academy - API Structure & Endpoints

## Overview

The Peer Academy API is a comprehensive RESTful service that provides access to all platform functionality. The API is organized into logical modules that correspond to the major system components, with consistent patterns for authentication, error handling, and data formatting.

## API Architecture

### Base Configuration
```typescript
const API_BASE_URL = 'https://api.peeracademy.org/v1';
const WS_BASE_URL = 'wss://api.peeracademy.org/v1/ws';

// API Versioning Strategy
interface APIVersion {
  version: string;
  releaseDate: Date;
  deprecationDate?: Date;
  supportedUntil: Date;
  breaking_changes: string[];
}

// Standard Response Format
interface APIResponse<T> {
  success: boolean;
  data?: T;
  error?: APIError;
  meta?: ResponseMetadata;
  pagination?: PaginationData;
}

interface APIError {
  code: string;
  message: string;
  details?: any;
  field?: string; // For validation errors
  timestamp: Date;
  requestId: string;
}

interface ResponseMetadata {
  requestId: string;
  timestamp: Date;
  executionTime: number;
  version: string;
  rateLimit?: RateLimitInfo;
}

interface PaginationData {
  page: number;
  limit: number;
  totalPages: number;
  totalItems: number;
  hasNext: boolean;
  hasPrevious: boolean;
}
```

### Authentication & Authorization

#### Authentication Endpoints
```typescript
// POST /auth/register
interface RegisterRequest {
  email: string;
  password: string;
  profile: {
    firstName: string;
    lastName: string;
    bio?: string;
  };
}

interface RegisterResponse {
  user: UserProfile;
  tokens: {
    accessToken: string;
    refreshToken: string;
    expiresAt: Date;
  };
  emailVerificationRequired: boolean;
}

// POST /auth/login
interface LoginRequest {
  email: string;
  password: string;
  rememberMe?: boolean;
}

interface LoginResponse {
  user: UserProfile;
  tokens: {
    accessToken: string;
    refreshToken: string;
    expiresAt: Date;
  };
  permissions: string[];
  votingWeights: VotingWeight[];
}

// POST /auth/refresh
interface RefreshRequest {
  refreshToken: string;
}

interface RefreshResponse {
  accessToken: string;
  expiresAt: Date;
}

// POST /auth/logout
interface LogoutRequest {
  refreshToken: string;
}

// GET /auth/verify-email/:token
interface EmailVerificationResponse {
  verified: boolean;
  user: UserProfile;
}

// Authentication Headers
const authHeaders = {
  'Authorization': 'Bearer <access_token>',
  'X-API-Key': '<api_key>', // For service-to-service calls
  'X-Request-ID': '<unique_request_id>'
};
```

#### Authorization Middleware
```typescript
// Role-based and context-aware authorization
interface AuthContext {
  user: UserProfile;
  permissions: Permission[];
  votingWeights: VotingWeight[];
  activeRoles: Role[];
  subjectMemberships: SubjectMembership[];
  settlementMemberships: SettlementMembership[];
}

// Authorization decorators for endpoints
const requireAuth = (permissions?: Permission[]) => { /* middleware */ };
const requireRole = (roles: string[]) => { /* middleware */ };
const requireSubjectMember = (subjectId?: string) => { /* middleware */ };
const requireSettlementMember = (settlementId?: string) => { /* middleware */ };
const requireMinimumScore = (minScore: number) => { /* middleware */ };
```

### User Management API

#### User Profile Endpoints
```typescript
// GET /users/profile
interface GetProfileResponse {
  user: UserProfile;
  stats: UserStats;
  achievements: Achievement[];
}

// PUT /users/profile
interface UpdateProfileRequest {
  firstName?: string;
  lastName?: string;
  bio?: string;
  avatar?: string;
  location?: string;
  preferences?: UserPreferences;
}

// GET /users/:userId
interface GetUserResponse {
  user: PublicUserProfile;
  stats: PublicUserStats;
  recentActivity: Activity[];
}

// GET /users/:userId/scores
interface GetUserScoresResponse {
  totalScore: number;
  subjectScores: SubjectScore[];
  scoreHistory: ScoreHistoryPoint[];
  ranking: UserRanking;
}

// POST /users/avatar/upload
interface UploadAvatarRequest {
  file: File; // multipart/form-data
}

interface UploadAvatarResponse {
  avatarUrl: string;
  thumbnailUrl: string;
}
```

### Subject Management API

#### Subject Endpoints
```typescript
// GET /subjects
interface GetSubjectsRequest {
  page?: number;
  limit?: number;
  search?: string;
  category?: string;
  settlementId?: string;
}

interface GetSubjectsResponse {
  subjects: SubjectSummary[];
  categories: string[];
  totalCount: number;
}

// GET /subjects/:subjectId
interface GetSubjectResponse {
  subject: Subject;
  members: SubjectMemberSummary[];
  recentPresentations: PresentationSummary[];
  statistics: SubjectStatistics;
  constitution: SubjectConstitution;
}

// POST /subjects
interface CreateSubjectRequest {
  name: string;
  description: string;
  category: string;
  parentSubjectId?: string;
  settlementId?: string;
  initialRules?: LocalRule[];
}

// POST /subjects/:subjectId/join
interface JoinSubjectRequest {
  motivation: string;
}

interface JoinSubjectResponse {
  membership: SubjectMembership;
  welcomeMessage: string;
  nextSteps: string[];
}

// DELETE /subjects/:subjectId/leave
interface LeaveSubjectResponse {
  success: boolean;
  effectiveDate: Date;
  dataRetention: string;
}

// GET /subjects/:subjectId/members
interface GetSubjectMembersRequest {
  page?: number;
  limit?: number;
  sortBy?: 'score' | 'joinDate' | 'name';
  order?: 'asc' | 'desc';
}

interface GetSubjectMembersResponse {
  members: SubjectMemberDetail[];
  representative: SubjectMemberDetail;
  totalMembers: number;
  activeMembers: number;
}
```

### Presentation System API

#### Presentation Management
```typescript
// POST /presentations/upload
interface UploadPresentationRequest {
  title: string;
  description: string;
  subjectId: string;
  contentType: 'video' | 'document' | 'slides' | 'mixed_media';
  files: File[]; // multipart/form-data
  metadata?: PresentationMetadata;
  isPublic?: boolean;
  monetizable?: boolean;
}

interface UploadPresentationResponse {
  presentation: Presentation;
  uploadStatus: UploadStatus;
  processingEstimate: string;
}

// GET /presentations
interface GetPresentationsRequest {
  page?: number;
  limit?: number;
  subjectId?: string;
  presenterId?: string;
  cycle?: number;
  year?: number;
  status?: PresentationStatus;
  sortBy?: 'score' | 'date' | 'views';
}

interface GetPresentationsResponse {
  presentations: PresentationSummary[];
  cycles: CycleSummary[];
  filters: FilterOptions;
}

// GET /presentations/:presentationId
interface GetPresentationResponse {
  presentation: PresentationDetail;
  evaluations: EvaluationSummary[];
  analytics: PresentationAnalytics;
  relatedPresentations: PresentationSummary[];
  accessInfo: AccessInfo;
}

// PUT /presentations/:presentationId
interface UpdatePresentationRequest {
  title?: string;
  description?: string;
  isPublic?: boolean;
  monetizable?: boolean;
  status?: PresentationStatus;
}

// DELETE /presentations/:presentationId
interface DeletePresentationResponse {
  success: boolean;
  deletionDate: Date;
  dataRetention: string;
}

// GET /presentations/:presentationId/download
// Returns file stream with appropriate headers
interface DownloadPresentationHeaders {
  'Content-Type': string;
  'Content-Disposition': string;
  'Content-Length': string;
  'Cache-Control': string;
}
```

#### Presentation Evaluation
```typescript
// POST /presentations/:presentationId/evaluate
interface SubmitEvaluationRequest {
  scores: {
    overall: number;
    content: number;
    clarity: number;
    originality: number;
    relevance: number;
    engagement: number;
  };
  feedback: {
    strengths: string;
    improvements: string;
    questions: string;
    publicComments?: string;
  };
  timeSpent: number;
  isAnonymous?: boolean;
}

interface SubmitEvaluationResponse {
  evaluation: PresentationEvaluation;
  presentationScore: number;
  userVotingWeight: number;
  rankingChange?: RankingChange;
}

// GET /presentations/:presentationId/evaluations
interface GetEvaluationsRequest {
  includeDetails?: boolean;
  includeAnonymous?: boolean;
  sortBy?: 'score' | 'date' | 'weight';
}

interface GetEvaluationsResponse {
  evaluations: EvaluationDetail[];
  statistics: EvaluationStatistics;
  averageScores: CategoryScores;
}

// PUT /presentations/:presentationId/evaluations/:evaluationId
interface UpdateEvaluationRequest {
  scores?: CategoryScores;
  feedback?: EvaluationFeedback;
}
```

### Governance & Voting API

#### Constitutional Management
```typescript
// GET /governance/constitution
interface GetConstitutionResponse {
  constitution: Constitution;
  activeRules: ConstitutionalRule[];
  pendingChanges: RuleChange[];
  votingHistory: VotingHistory[];
}

// GET /governance/rules/:ruleId
interface GetRuleResponse {
  rule: ConstitutionalRule;
  votes: RuleVote[];
  modifications: RuleModification[];
  dependencies: ConstitutionalRule[];
}

// POST /governance/rules/:ruleId/vote
interface VoteOnRuleRequest {
  voteType: 'strengthen' | 'weaken' | 'remove';
  reasoning: string;
  cycle: number;
}

interface VoteOnRuleResponse {
  vote: RuleVote;
  currentRuleValue: number;
  votingWeight: number;
  voteAccepted: boolean;
}

// POST /governance/rules
interface ProposeRuleRequest {
  title: string;
  description: string;
  category: RuleCategory;
  initialValue: number;
  votingThreshold: number;
  dependencies?: string[];
}

interface ProposeRuleResponse {
  rule: ConstitutionalRule;
  status: 'pending_approval' | 'active';
  creditCost: number;
}
```

#### Colloquium Operations
```typescript
// GET /governance/colloquium
interface GetColloguiumResponse {
  colloquium: Colloquium;
  currentMembers: ColloguiumMember[];
  upcomingSessions: ColloguiumSession[];
  recentDecisions: ColloguiumDecision[];
}

// POST /governance/colloquium/sessions
interface CreateSessionRequest {
  agenda: AgendaItem[];
  scheduledTime: Date;
  duration: number;
}

interface CreateSessionResponse {
  session: ColloguiumSession;
  notifications: NotificationSummary[];
}

// POST /governance/colloquium/sessions/:sessionId/vote
interface ColloguiumVoteRequest {
  agendaItemId: string;
  position: 'support' | 'oppose' | 'abstain';
  reasoning?: string;
}

interface ColloguiumVoteResponse {
  vote: ColloguiumVote;
  currentTally: VoteTally;
  quorumMet: boolean;
}
```

#### Director Powers
```typescript
// POST /governance/director/actions
interface DirectorActionRequest {
  actionType: DirectorPowerType;
  data: any;
  justification: string;
}

interface DirectorActionResponse {
  action: DirectorAction;
  creditUsed: number;
  requiresApproval: boolean;
  effectiveDate?: Date;
}

// POST /governance/director/emergency
interface ActivateEmergencyRequest {
  trigger: EmergencyTrigger;
  justification: string;
  duration?: number;
}

interface ActivateEmergencyResponse {
  activated: boolean;
  expirationDate: Date;
  enhancedPowers: DirectorPowerType[];
  notificationsSent: number;
}
```

### Philosophy & Proposition API

#### Proposition Management
```typescript
// GET /philosophy/branches
interface GetBranchesResponse {
  branches: PhilosophyBranch[];
  crossBranchConnections: BranchConnection[];
  statistics: PhilosophyStatistics;
}

// GET /philosophy/propositions
interface GetPropositionsRequest {
  branchId?: string;
  status?: PropositionStatus[];
  minValue?: number;
  maxValue?: number;
  search?: string;
  sortBy?: 'value' | 'created' | 'modified';
}

interface GetPropositionsResponse {
  propositions: PropositionSummary[];
  facets: SearchFacets;
  trends: PropositionTrends;
}

// POST /philosophy/propositions
interface CreatePropositionRequest {
  branchId: string;
  title: string;
  statement: string;
  evidence: Evidence[];
  reasoning: string;
  scope: PropositionScope;
  logicalForm: LogicalForm;
  dependencies?: string[];
}

interface CreatePropositionResponse {
  proposition: Proposition;
  conflicts: ConflictCheck[];
  requiresReview: boolean;
}

// POST /philosophy/propositions/:propositionId/vote
interface VoteOnPropositionRequest {
  voteType: 'support' | 'oppose' | 'strengthen' | 'weaken';
  reasoning: string;
  evidence?: Evidence[];
}

interface VoteOnPropositionResponse {
  vote: PropositionVote;
  newValue: number;
  statusChange?: PropositionStatus;
}

// GET /philosophy/synthesis
interface GetPhilosophySynthesisRequest {
  version?: number;
  includeDetails?: boolean;
}

interface GetPhilosophySynthesisResponse {
  synthesis: PhilosophicalFramework;
  branches: BranchSummary[];
  knowledgeGaps: KnowledgeGap[];
}
```

#### Logic and Consistency
```typescript
// POST /philosophy/consistency/check
interface CheckConsistencyRequest {
  branchId?: string;
  propositionIds?: string[];
}

interface CheckConsistencyResponse {
  report: ConsistencyReport;
  conflicts: ConflictGroup[];
  recommendations: string[];
}

// GET /philosophy/search
interface PhilosophySearchRequest {
  query: string;
  branches?: string[];
  status?: PropositionStatus[];
  searchType?: 'text' | 'semantic' | 'logical';
}

interface PhilosophySearchResponse {
  results: SearchResult[];
  suggestions: string[];
  relatedConcepts: Concept[];
}
```

### Settlement Management API

#### Settlement Operations
```typescript
// GET /settlements
interface GetSettlementsRequest {
  page?: number;
  limit?: number;
  status?: SettlementStatus[];
  region?: string;
  hasSpace?: boolean;
}

interface GetSettlementsResponse {
  settlements: SettlementSummary[];
  regions: RegionSummary[];
  networkStatistics: NetworkStatistics;
}

// GET /settlements/:settlementId
interface GetSettlementResponse {
  settlement: SettlementDetail;
  members: SettlementMemberSummary[];
  resources: ResourceSummary[];
  governance: SettlementGovernance;
  analytics: SettlementAnalytics;
}

// POST /settlements/propose
interface ProposeSettlementRequest {
  name: string;
  description: string;
  location: SettlementLocation;
  plannedCapacity: number;
  subjects: string[];
  economicModel: EconomicModel;
  fundingPlan: FundingPlan;
  timeline: SettlementTimeline;
}

interface ProposeSettlementResponse {
  proposal: SettlementProposal;
  estimatedReviewTime: string;
  reviewers: ReviewerInfo[];
}

// POST /settlements/:settlementId/join
interface JoinSettlementRequest {
  residencyType: 'resident' | 'visitor';
  plannedDuration?: string;
  motivation: string;
  skills?: string[];
}

interface JoinSettlementResponse {
  membership: SettlementMembership;
  orientation: OrientationInfo;
  housing?: HousingAssignment;
}
```

#### Resource Management
```typescript
// GET /settlements/:settlementId/resources
interface GetResourcesRequest {
  type?: ResourceType[];
  availability?: 'available' | 'reserved' | 'unavailable';
  includeShared?: boolean;
}

interface GetResourcesResponse {
  resources: Resource[];
  utilization: ResourceUtilization;
  sharing: ResourceSharing[];
}

// POST /settlements/:settlementId/resources/request
interface ResourceRequestRequest {
  resourceType: ResourceType;
  amount: number;
  duration: string;
  justification: string;
  priority: RequestPriority;
  deadline?: Date;
}

interface ResourceRequestResponse {
  request: ResourceRequest;
  estimatedResponse: string;
  alternatives: ResourceAlternative[];
}

// GET /settlements/network/coordination
interface NetworkCoordinationResponse {
  activeExchanges: ResourceExchange[];
  upcomingEvents: NetworkEvent[];
  collaborations: InterSettlementCollaboration[];
  opportunities: CollaborationOpportunity[];
}
```

### Monetization & Payment API

#### Content Monetization
```typescript
// POST /monetization/enable
interface EnableMonetizationRequest {
  paymentMethods: PaymentMethodSetup[];
  enabledStreams: MonetizationStream[];
  preferences: MonetizationPreferences;
}

interface EnableMonetizationResponse {
  profile: CreatorMonetizationProfile;
  accountId: string;
  projectedEarnings: EarningsProjection;
}

// POST /presentations/:presentationId/monetize
interface MonetizePresentationRequest {
  model: MonetizationModel;
  pricing: PricingConfig;
  accessLevel: AccessLevel;
  restrictions?: AccessRestrictions;
}

interface MonetizePresentationResponse {
  monetization: PresentationMonetization;
  estimatedRevenue: RevenueEstimate;
  marketingSuggestions: string[];
}

// GET /monetization/analytics
interface GetMonetizationAnalyticsRequest {
  timeRange: TimeRange;
  breakdown?: 'daily' | 'weekly' | 'monthly';
}

interface GetMonetizationAnalyticsResponse {
  revenue: RevenueBreakdown;
  trends: RevenueTrends;
  topContent: ContentPerformance[];
}
```

#### Course Management
```typescript
// POST /courses
interface CreateCourseRequest {
  title: string;
  description: string;
  subjectId: string;
  curriculum: CourseModule[];
  pricing: CoursePricing;
  difficulty: CourseDifficulty;
  maxEnrollments: number;
  prerequisites?: string[];
}

interface CreateCourseResponse {
  course: Course;
  setupSteps: string[];
  estimatedLaunch: Date;
}

// POST /courses/:courseId/enroll
interface EnrollInCourseRequest {
  paymentMethodId?: string;
  couponCode?: string;
}

interface EnrollInCourseResponse {
  enrollment: CourseEnrollment;
  accessUrl: string;
  startingModule: CourseModule;
  certificateEligible: boolean;
}

// GET /courses/:courseId/progress
interface GetCourseProgressResponse {
  progress: CourseProgress;
  completedModules: string[];
  currentModule: CourseModule;
  estimatedCompletion: Date;
  certificateStatus: CertificateStatus;
}
```

#### Fundraising
```typescript
// POST /fundraising/campaigns
interface CreateCampaignRequest {
  title: string;
  description: string;
  targetAmount: number;
  currency: string;
  scope: CampaignScope;
  purpose: CampaignPurpose;
  endDate: Date;
  contributionTiers: ContributionTier[];
}

interface CreateCampaignResponse {
  campaign: FundraisingCampaign;
  requiresApproval: boolean;
  promotionSuggestions: string[];
}

// POST /fundraising/campaigns/:campaignId/contribute
interface ContributeRequest {
  amount: number;
  paymentMethodId: string;
  message?: string;
  isAnonymous?: boolean;
}

interface ContributeResponse {
  contribution: CampaignContribution;
  rewards: ContributionReward[];
  totalRaised: number;
  percentOfGoal: number;
}

// GET /fundraising/campaigns/:campaignId/updates
interface GetCampaignUpdatesResponse {
  updates: CampaignUpdate[];
  milestones: CampaignMilestone[];
  impactReport?: ImpactReport;
}
```

### Search & Analytics API

#### Universal Search
```typescript
// GET /search
interface UniversalSearchRequest {
  query: string;
  filters?: SearchFilters;
  scope?: SearchScope[];
  sortBy?: 'relevance' | 'date' | 'score';
  limit?: number;
}

interface UniversalSearchResponse {
  results: SearchResult[];
  suggestions: string[];
  facets: SearchFacets;
  totalResults: number;
}

interface SearchResult {
  type: 'user' | 'presentation' | 'proposition' | 'course' | 'settlement';
  id: string;
  title: string;
  description: string;
  url: string;
  score: number;
  highlights: string[];
  metadata: SearchMetadata;
}

// GET /search/suggestions
interface SearchSuggestionsRequest {
  query: string;
  context?: string;
}

interface SearchSuggestionsResponse {
  suggestions: string[];
  relatedTopics: string[];
  popularSearches: string[];
}
```

#### Analytics
```typescript
// GET /analytics/dashboard
interface GetDashboardRequest {
  timeRange: TimeRange;
  metrics?: string[];
}

interface GetDashboardResponse {
  overview: DashboardOverview;
  trends: TrendData[];
  topContent: ContentMetrics[];
  engagement: EngagementMetrics;
  growth: GrowthMetrics;
}

// GET /analytics/users/:userId
interface GetUserAnalyticsRequest {
  userId: string;
  includePrivate?: boolean; // Requires permission
  timeRange?: TimeRange;
}

interface GetUserAnalyticsResponse {
  performance: UserPerformance;
  activity: ActivitySummary;
  contributions: ContributionSummary;
  growth: UserGrowthMetrics;
}

// GET /analytics/subjects/:subjectId
interface GetSubjectAnalyticsResponse {
  engagement: SubjectEngagement;
  performance: SubjectPerformance;
  memberGrowth: MembershipTrends;
  contentMetrics: ContentMetrics;
}
```

### WebSocket API

#### Real-time Features
```typescript
// WebSocket Connection
const ws = new WebSocket(`${WS_BASE_URL}?token=${accessToken}`);

// Event Types
interface WebSocketMessage {
  type: string;
  channel?: string;
  data: any;
  timestamp: Date;
  messageId: string;
}

// Subscription Management
interface SubscribeMessage {
  type: 'subscribe';
  channels: string[];
}

interface UnsubscribeMessage {
  type: 'unsubscribe';
  channels: string[];
}

// Real-time Events
interface VoteUpdateEvent {
  type: 'vote_update';
  channel: 'governance' | 'proposition' | 'presentation';
  data: {
    itemId: string;
    newScore?: number;
    voteCount: number;
    trend: 'increasing' | 'decreasing' | 'stable';
  };
}

interface NotificationEvent {
  type: 'notification';
  channel: 'user_notifications';
  data: {
    id: string;
    type: NotificationType;
    title: string;
    message: string;
    actionUrl?: string;
    priority: 'low' | 'medium' | 'high' | 'urgent';
  };
}

interface LiveSessionEvent {
  type: 'live_session';
  channel: string; // 'presentation_123', 'colloquium_456'
  data: {
    action: 'join' | 'leave' | 'vote' | 'comment' | 'sync';
    userId?: string;
    payload: any;
  };
}
```

## Error Handling

### Standard Error Codes
```typescript
enum APIErrorCode {
  // Authentication & Authorization
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  INVALID_CREDENTIALS = 'INVALID_CREDENTIALS',
  
  // Validation
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INVALID_INPUT = 'INVALID_INPUT',
  MISSING_REQUIRED_FIELD = 'MISSING_REQUIRED_FIELD',
  
  // Resource Management
  RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND',
  RESOURCE_ALREADY_EXISTS = 'RESOURCE_ALREADY_EXISTS',
  RESOURCE_CONFLICT = 'RESOURCE_CONFLICT',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',
  
  // Business Logic
  INSUFFICIENT_SCORE = 'INSUFFICIENT_SCORE',
  VOTING_PERIOD_CLOSED = 'VOTING_PERIOD_CLOSED',
  ALREADY_EVALUATED = 'ALREADY_EVALUATED',
  CAPACITY_EXCEEDED = 'CAPACITY_EXCEEDED',
  
  // System Errors
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
  
  // Payment Errors
  PAYMENT_FAILED = 'PAYMENT_FAILED',
  INSUFFICIENT_FUNDS = 'INSUFFICIENT_FUNDS',
  PAYMENT_METHOD_INVALID = 'PAYMENT_METHOD_INVALID'
}

// Error Response Examples
const errorResponses = {
  validation: {
    success: false,
    error: {
      code: 'VALIDATION_ERROR',
      message: 'Invalid input data',
      details: {
        fields: {
          email: 'Invalid email format',
          password: 'Password must be at least 12 characters'
        }
      },
      timestamp: new Date(),
      requestId: 'req_123456'
    }
  },
  
  unauthorized: {
    success: false,
    error: {
      code: 'UNAUTHORIZED',
      message: 'Authentication required',
      timestamp: new Date(),
      requestId: 'req_123456'
    }
  },
  
  forbidden: {
    success: false,
    error: {
      code: 'INSUFFICIENT_SCORE',
      message: 'Minimum score of 100 required for this action',
      details: {
        required: 100,
        current: 75,
        action: 'create_proposition'
      },
      timestamp: new Date(),
      requestId: 'req_123456'
    }
  }
};
```

## Rate Limiting

### Rate Limit Configuration
```typescript
interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  skipSuccessfulRequests?: boolean;
  keyGenerator?: (req: Request) => string;
  onLimitReached?: (req: Request) => void;
}

const rateLimits = {
  // Global API limits
  global: { windowMs: 15 * 60 * 1000, maxRequests: 1000 },
  
  // Authentication endpoints (stricter)
  auth: { windowMs: 15 * 60 * 1000, maxRequests: 5 },
  
  // Content upload (very limited)
  upload: { windowMs: 60 * 60 * 1000, maxRequests: 10 },
  
  // Search endpoints
  search: { windowMs: 60 * 1000, maxRequests: 60 },
  
  // Payment processing (strict)
  payment: { windowMs: 60 * 60 * 1000, maxRequests: 20 }
};

// Rate limit headers
interface RateLimitHeaders {
  'X-RateLimit-Limit': string;
  'X-RateLimit-Remaining': string;
  'X-RateLimit-Reset': string;
  'X-RateLimit-Policy': string;
}
```

## API Documentation

### OpenAPI Specification
```yaml
openapi: 3.0.3
info:
  title: Peer Academy API
  description: Comprehensive API for the Peer Academy educational platform
  version: 1.0.0
  contact:
    name: API Support
    url: https://docs.peeracademy.org
    email: api-support@peeracademy.org
servers:
  - url: https://api.peeracademy.org/v1
    description: Production server
  - url: https://staging-api.peeracademy.org/v1
    description: Staging server
paths:
  /auth/login:
    post:
      tags:
        - Authentication
      summary: User login
      description: Authenticate user and return access tokens
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LoginRequest'
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LoginResponse'
        '401':
          description: Invalid credentials
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```

### SDK Generation
```typescript
// Auto-generated SDK for different languages
interface SDKOptions {
  language: 'typescript' | 'python' | 'java' | 'go' | 'rust';
  outputDir: string;
  packageName: string;
  version: string;
}

// TypeScript SDK example
class PeerAcademyAPI {
  private baseURL: string;
  private accessToken?: string;
  
  auth = new AuthAPI(this);
  users = new UsersAPI(this);
  subjects = new SubjectsAPI(this);
  presentations = new PresentationsAPI(this);
  governance = new GovernanceAPI(this);
  philosophy = new PhilosophyAPI(this);
  settlements = new SettlementsAPI(this);
  monetization = new MonetizationAPI(this);
  search = new SearchAPI(this);
  
  constructor(options: SDKOptions) {
    this.baseURL = options.baseURL;
  }
  
  setAccessToken(token: string) {
    this.accessToken = token;
  }
}

// Usage example
const api = new PeerAcademyAPI({ baseURL: 'https://api.peeracademy.org/v1' });
await api.auth.login({ email: 'user@example.com', password: 'password123' });
const presentations = await api.presentations.list({ subjectId: 'math-101' });
```

This comprehensive API structure provides a robust, scalable interface for all Peer Academy platform functionality, with consistent patterns for authentication, error handling, and data exchange across all system components.