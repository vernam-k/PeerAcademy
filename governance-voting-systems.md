# Peer Academy - Governance & Voting Systems

## Overview

The governance system implements the sophisticated meritocratic framework. It combines weighted voting based on merit scores, evolutionary rule systems, director powers with constraints, and colloquium decision-making. The system must balance stability with adaptability while preventing both mob rule and authoritarian takeover.

## Constitutional Framework

### Core Constitutional Structure
```typescript
interface Constitution {
  id: string;
  version: number;
  scope: 'academy' | 'subject' | 'settlement';
  scopeId?: string;
  foundingPrinciples: FoundingPrinciple[];
  rules: ConstitutionalRule[];
  amendments: Amendment[];
  lastModified: Date;
  ratificationDate: Date;
  nextReviewDate: Date;
}

interface FoundingPrinciple {
  id: string;
  title: string;
  description: string;
  immutable: boolean; // Cannot be changed
  order: number;
}

interface ConstitutionalRule {
  id: string;
  title: string;
  description: string;
  category: RuleCategory;
  value: number; // 0-100, higher values are harder to modify/remove
  votingThreshold: number; // Required majority to modify (0.5-0.9)
  createdBy: string; // userId or 'founding'
  createdAt: Date;
  lastModified: Date;
  modificationHistory: RuleModification[];
  dependencies: string[]; // other rule IDs this depends on
  conflicts: string[]; // rules that conflict with this one
}

enum RuleCategory {
  FUNDAMENTAL_RIGHTS = 'fundamental_rights',
  GOVERNANCE_STRUCTURE = 'governance_structure', 
  ACADEMIC_STANDARDS = 'academic_standards',
  CONDUCT_STANDARDS = 'conduct_standards',
  SETTLEMENT_OPERATIONS = 'settlement_operations',
  EMERGENCY_PROVISIONS = 'emergency_provisions'
}

class ConstitutionalEngine {
  /**
   * Initialize academy constitution with founding principles
   */
  async initializeConstitution(
    scope: string,
    scopeId?: string,
    foundingPrinciples?: FoundingPrinciple[]
  ): Promise<Constitution> {
    
    const constitution: Constitution = {
      id: generateId(),
      version: 1,
      scope: scope as any,
      scopeId,
      foundingPrinciples: foundingPrinciples || this.getDefaultPrinciples(),
      rules: await this.createFoundingRules(),
      amendments: [],
      ratificationDate: new Date(),
      lastModified: new Date(),
      nextReviewDate: this.addMonths(new Date(), 6) // Review every 6 months
    };
    
    await this.saveConstitution(constitution);
    return constitution;
  }
  
  private getDefaultPrinciples(): FoundingPrinciple[] {
    return [
      {
        id: '1',
        title: 'Freedom of Speech and Inquiry',
        description: 'All members have the right to express ideas and pursue knowledge without fear of persecution',
        immutable: true,
        order: 1
      },
      {
        id: '2', 
        title: 'Merit-Based Recognition',
        description: 'Authority and influence are earned through demonstrated competence and contribution',
        immutable: true,
        order: 2
      },
      {
        id: '3',
        title: 'Transparent Governance',
        description: 'All governance processes and decisions must be transparent and accountable',
        immutable: true,
        order: 3
      },
      {
        id: '4',
        title: 'Continuous Learning',
        description: 'The academy exists to advance knowledge and human flourishing through education',
        immutable: true,
        order: 4
      }
    ];
  }
  
  private async createFoundingRules(): Promise<ConstitutionalRule[]> {
    return [
      {
        id: 'rule-001',
        title: 'Freedom of Academic Expression',
        description: 'No member may be penalized for presenting ideas or research that challenges conventional wisdom',
        category: RuleCategory.FUNDAMENTAL_RIGHTS,
        value: 95, // Very high value, extremely hard to remove
        votingThreshold: 0.85,
        createdBy: 'founding',
        createdAt: new Date(),
        lastModified: new Date(),
        modificationHistory: [],
        dependencies: [],
        conflicts: []
      },
      {
        id: 'rule-002',
        title: 'Weighted Voting System',
        description: 'Voting power is determined by demonstrated competence through peer evaluation',
        category: RuleCategory.GOVERNANCE_STRUCTURE,
        value: 90,
        votingThreshold: 0.75,
        createdBy: 'founding',
        createdAt: new Date(),
        lastModified: new Date(),
        modificationHistory: [],
        dependencies: [],
        conflicts: []
      },
      {
        id: 'rule-003',
        title: 'Director Term Limits',
        description: 'No individual may serve as Director for more than two consecutive cycles',
        category: RuleCategory.GOVERNANCE_STRUCTURE,
        value: 80,
        votingThreshold: 0.70,
        createdBy: 'founding',
        createdAt: new Date(),
        lastModified: new Date(),
        modificationHistory: [],
        dependencies: ['rule-002'],
        conflicts: []
      }
    ];
  }
}
```

### Rule Evolution Engine
```typescript
class RuleEvolutionEngine {
  private currentCycle: number;
  
  /**
   * Process rule modifications for a governance cycle
   */
  async processCycleVotes(
    constitutionId: string,
    cycle: number
  ): Promise<RuleEvolutionResult> {
    
    this.currentCycle = cycle;
    const constitution = await this.getConstitution(constitutionId);
    const votes = await this.getCycleVotes(constitutionId, cycle);
    
    const results: RuleModificationResult[] = [];
    
    for (const rule of constitution.rules) {
      const ruleVotes = votes.filter(v => v.ruleId === rule.id);
      if (ruleVotes.length === 0) continue;
      
      const result = await this.processRuleVotes(rule, ruleVotes);
      results.push(result);
      
      if (result.modified) {
        await this.updateRule(rule.id, result);
      }
    }
    
    // Check for new rules proposed by Director
    const newRules = await this.processDirectorRules(constitutionId, cycle);
    results.push(...newRules);
    
    await this.recordCycleResults(constitutionId, cycle, results);
    
    return {
      cycle,
      rulesModified: results.filter(r => r.modified).length,
      rulesCreated: newRules.length,
      rulesRemoved: results.filter(r => r.removed).length,
      details: results
    };
  }
  
  /**
   * Process votes for a specific rule
   */
  private async processRuleVotes(
    rule: ConstitutionalRule,
    votes: RuleVote[]
  ): Promise<RuleModificationResult> {
    
    // Calculate weighted vote totals
    let strengthenWeight = 0;
    let weakenWeight = 0;
    let removeWeight = 0;
    let totalWeight = 0;
    
    votes.forEach(vote => {
      totalWeight += vote.votingWeight;
      switch (vote.voteType) {
        case 'strengthen':
          strengthenWeight += vote.votingWeight;
          break;
        case 'weaken':
          weakenWeight += vote.votingWeight;
          break;
        case 'remove':
          removeWeight += vote.votingWeight;
          break;
      }
    });
    
    // Calculate required thresholds
    const requiredMajority = totalWeight * rule.votingThreshold;
    const removalThreshold = totalWeight * 0.67; // 2/3 majority to remove
    
    let newValue = rule.value;
    let removed = false;
    let modified = false;
    
    // Check for removal first
    if (removeWeight >= removalThreshold && rule.value < 50) {
      removed = true;
      modified = true;
    } else {
      // Calculate value change
      const netStrengthening = strengthenWeight - weakenWeight;
      const strengthenRatio = netStrengthening / totalWeight;
      
      // Value change is proportional to consensus and current value
      const maxChange = this.calculateMaxValueChange(rule.value, strengthenRatio);
      const valueChange = maxChange * Math.abs(strengthenRatio);
      
      if (Math.abs(valueChange) >= 1) { // Only record significant changes
        newValue = Math.max(0, Math.min(100, rule.value + 
          (strengthenRatio > 0 ? valueChange : -valueChange)));
        modified = true;
      }
    }
    
    return {
      ruleId: rule.id,
      oldValue: rule.value,
      newValue,
      removed,
      modified,
      voteDetails: {
        totalVotes: votes.length,
        totalWeight,
        strengthenWeight,
        weakenWeight,
        removeWeight,
        consensus: this.calculateConsensus(votes)
      }
    };
  }
  
  /**
   * Calculate maximum allowed value change based on current value
   */
  private calculateMaxValueChange(currentValue: number, strengthenRatio: number): number {
    // Higher value rules are more resistant to change
    const resistance = Math.pow(currentValue / 100, 2);
    const baseChange = 10; // Base maximum change per cycle
    
    return baseChange * (1 - resistance * 0.8); // Up to 80% resistance
  }
  
  /**
   * Process new rules proposed by the Director
   */
  private async processDirectorRules(
    constitutionId: string,
    cycle: number
  ): Promise<RuleModificationResult[]> {
    
    const director = await this.getCurrentDirector(constitutionId);
    if (!director) return [];
    
    const directorCredit = await this.getDirectorCredit(director.id, cycle);
    const proposedRules = await this.getDirectorProposedRules(director.id, cycle);
    
    const results: RuleModificationResult[] = [];
    let creditUsed = 0;
    
    for (const proposedRule of proposedRules) {
      const ruleCost = this.calculateRuleCost(proposedRule);
      
      if (creditUsed + ruleCost <= directorCredit.available) {
        // Validate rule doesn't conflict with existing rules
        const validationResult = await this.validateNewRule(
          constitutionId, 
          proposedRule
        );
        
        if (validationResult.valid) {
          // Create the rule, but requires colloquium approval
          const pendingRule = await this.createPendingRule(
            proposedRule,
            director.id,
            cycle
          );
          
          creditUsed += ruleCost;
          results.push({
            ruleId: pendingRule.id,
            oldValue: 0,
            newValue: proposedRule.initialValue,
            removed: false,
            modified: true,
            isNew: true,
            requiresApproval: true
          });
        } else {
          results.push({
            ruleId: proposedRule.id,
            oldValue: 0,
            newValue: 0,
            removed: false,
            modified: false,
            rejected: true,
            rejectionReason: validationResult.reason
          });
        }
      } else {
        results.push({
          ruleId: proposedRule.id,
          oldValue: 0,
          newValue: 0,
          removed: false,
          modified: false,
          rejected: true,
          rejectionReason: 'Insufficient director credit'
        });
      }
    }
    
    await this.updateDirectorCredit(director.id, cycle, creditUsed);
    return results;
  }
}

interface RuleVote {
  id: string;
  ruleId: string;
  voterId: string;
  voteType: 'strengthen' | 'weaken' | 'remove';
  votingWeight: number;
  cycle: number;
  reasoning?: string;
  castAt: Date;
}

interface RuleModificationResult {
  ruleId: string;
  oldValue: number;
  newValue: number;
  removed: boolean;
  modified: boolean;
  isNew?: boolean;
  requiresApproval?: boolean;
  rejected?: boolean;
  rejectionReason?: string;
  voteDetails?: {
    totalVotes: number;
    totalWeight: number;
    strengthenWeight: number;
    weakenWeight: number;
    removeWeight: number;
    consensus: number;
  };
}
```

## Colloquium Governance System

### Colloquium Structure and Operations
```typescript
interface Colloquium {
  id: string;
  constitutionId: string;
  currentCycle: number;
  members: ColloguiumMember[];
  director: Director | null;
  currentSession?: ColloguiumSession;
  decisionHistory: ColloguiumDecision[];
  votingRules: ColloguiumVotingRules;
}

interface ColloguiumMember {
  userId: string;
  subjectId: string;
  representingSubject: string;
  votingWeight: number;
  joinedCycle: number;
  status: 'active' | 'inactive' | 'suspended';
  attendanceRate: number;
}

interface ColloguiumSession {
  id: string;
  sessionNumber: number;
  cycle: number;
  agenda: AgendaItem[];
  startTime: Date;
  endTime?: Date;
  status: 'scheduled' | 'active' | 'completed' | 'cancelled';
  decisions: ColloguiumDecision[];
  attendees: string[];
}

interface AgendaItem {
  id: string;
  type: 'rule_vote' | 'director_election' | 'budget_allocation' | 'settlement_approval' | 'emergency_motion';
  title: string;
  description: string;
  proposedBy: string;
  requiredMajority: number;
  timeAllotted: number; // minutes
  supportingDocuments: string[];
  status: 'pending' | 'under_discussion' | 'voting' | 'decided';
}

class ColloguiumEngine {
  /**
   * Initialize a new colloquium session
   */
  async initializeSession(
    colloguiumId: string,
    cycle: number,
    agenda: AgendaItem[]
  ): Promise<ColloguiumSession> {
    
    const colloquium = await this.getColloquium(colloguiumId);
    
    // Validate quorum requirements
    const activeMembers = colloquium.members.filter(m => m.status === 'active');
    const quorumRequired = Math.ceil(activeMembers.length * 0.6); // 60% for quorum
    
    const session: ColloguiumSession = {
      id: generateId(),
      sessionNumber: await this.getNextSessionNumber(colloguiumId),
      cycle,
      agenda,
      startTime: new Date(),
      status: 'scheduled',
      decisions: [],
      attendees: []
    };
    
    await this.saveSession(session);
    
    // Notify all colloquium members
    await this.notifyMembers(colloquium.members, session);
    
    return session;
  }
  
  /**
   * Process a vote on a colloquium agenda item
   */
  async processColloguiumVote(
    sessionId: string,
    agendaItemId: string,
    votes: ColloguiumVote[]
  ): Promise<VotingResult> {
    
    const session = await this.getSession(sessionId);
    const agendaItem = session.agenda.find(item => item.id === agendaItemId);
    
    if (!agendaItem) {
      throw new Error('Agenda item not found');
    }
    
    // Calculate weighted vote totals
    let supportWeight = 0;
    let opposeWeight = 0;
    let abstainWeight = 0;
    let totalPossibleWeight = 0;
    
    const colloquium = await this.getColloquium(session.colloguiumId);
    const activeMembers = colloquium.members.filter(m => m.status === 'active');
    
    // Calculate total possible voting weight
    totalPossibleWeight = activeMembers.reduce((sum, member) => 
      sum + member.votingWeight, 0
    );
    
    // Process votes
    votes.forEach(vote => {
      const member = activeMembers.find(m => m.userId === vote.voterId);
      if (!member) return;
      
      switch (vote.position) {
        case 'support':
          supportWeight += member.votingWeight;
          break;
        case 'oppose':
          opposeWeight += member.votingWeight;
          break;
        case 'abstain':
          abstainWeight += member.votingWeight;
          break;
      }
    });
    
    const totalVotesWeight = supportWeight + opposeWeight + abstainWeight;
    
    // Check if quorum is met (60% of total weight must participate)
    const quorumMet = totalVotesWeight >= (totalPossibleWeight * 0.6);
    
    if (!quorumMet) {
      return {
        passed: false,
        reason: 'Quorum not met',
        supportWeight,
        opposeWeight,
        abstainWeight,
        totalVotesWeight,
        totalPossibleWeight,
        requiredMajority: agendaItem.requiredMajority
      };
    }
    
    // Calculate if motion passes
    const supportRatio = supportWeight / (supportWeight + opposeWeight);
    const passed = supportRatio >= agendaItem.requiredMajority;
    
    const decision: ColloguiumDecision = {
      id: generateId(),
      sessionId,
      agendaItemId,
      type: agendaItem.type,
      passed,
      supportWeight,
      opposeWeight,
      abstainWeight,
      requiredMajority: agendaItem.requiredMajority,
      actualMajority: supportRatio,
      votes,
      decidedAt: new Date(),
      effectiveDate: this.calculateEffectiveDate(agendaItem.type)
    };
    
    await this.recordDecision(decision);
    
    // Execute decision if it passed
    if (passed) {
      await this.executeColloguiumDecision(decision);
    }
    
    return {
      passed,
      supportWeight,
      opposeWeight,
      abstainWeight,
      totalVotesWeight,
      totalPossibleWeight,
      requiredMajority: agendaItem.requiredMajority,
      actualMajority: supportRatio
    };
  }
  
  /**
   * Execute a passed colloquium decision
   */
  private async executeColloguiumDecision(decision: ColloguiumDecision): Promise<void> {
    switch (decision.type) {
      case 'director_election':
        await this.executeDirectorElection(decision);
        break;
      case 'rule_vote':
        await this.executeRuleModification(decision);
        break;
      case 'settlement_approval':
        await this.executeSettlementApproval(decision);
        break;
      case 'budget_allocation':
        await this.executeBudgetAllocation(decision);
        break;
      case 'emergency_motion':
        await this.executeEmergencyMotion(decision);
        break;
    }
  }
}

interface ColloguiumVote {
  voterId: string;
  position: 'support' | 'oppose' | 'abstain';
  reasoning?: string;
  castAt: Date;
}

interface ColloguiumDecision {
  id: string;
  sessionId: string;
  agendaItemId: string;
  type: string;
  passed: boolean;
  supportWeight: number;
  opposeWeight: number;
  abstainWeight: number;
  requiredMajority: number;
  actualMajority: number;
  votes: ColloguiumVote[];
  decidedAt: Date;
  effectiveDate: Date;
}
```

## Director Powers and Constraints

### Director Authority System
```typescript
interface Director {
  userId: string;
  constitutionId: string;
  electedCycle: number;
  termStartDate: Date;
  termEndDate: Date;
  emergencyPowersActive: boolean;
  emergencyActivatedAt?: Date;
  powers: DirectorPower[];
  constraints: DirectorConstraint[];
  creditBalance: DirectorCredit;
  actions: DirectorAction[];
}

interface DirectorPower {
  type: DirectorPowerType;
  description: string;
  constraints: string[];
  usageLimit?: number;
  usageCount: number;
  lastUsed?: Date;
}

enum DirectorPowerType {
  CREATE_RULE = 'create_rule',
  VETO_RULE = 'veto_rule',
  EMERGENCY_OVERRIDE = 'emergency_override',
  BUDGET_ALLOCATION = 'budget_allocation',
  SETTLEMENT_APPROVAL = 'settlement_approval',
  MEMBER_SUSPENSION = 'member_suspension',
  CALL_EMERGENCY_SESSION = 'call_emergency_session'
}

interface DirectorConstraint {
  type: 'credit_limit' | 'time_limit' | 'approval_required' | 'emergency_only';
  value: any;
  description: string;
}

class DirectorEngine {
  /**
   * Execute director action with validation and constraints
   */
  async executeDirectorAction(
    directorId: string,
    actionType: DirectorPowerType,
    actionData: any
  ): Promise<DirectorActionResult> {
    
    const director = await this.getDirector(directorId);
    
    // Validate director has this power
    const power = director.powers.find(p => p.type === actionType);
    if (!power) {
      throw new Error('Director does not have this power');
    }
    
    // Check constraints
    const constraintCheck = await this.checkConstraints(director, actionType, actionData);
    if (!constraintCheck.allowed) {
      return {
        success: false,
        reason: constraintCheck.reason,
        actionId: null
      };
    }
    
    // Check usage limits
    if (power.usageLimit && power.usageCount >= power.usageLimit) {
      return {
        success: false,
        reason: 'Usage limit exceeded for this power',
        actionId: null
      };
    }
    
    // Execute the action
    const actionResult = await this.performDirectorAction(
      director, 
      actionType, 
      actionData
    );
    
    // Record the action
    const action: DirectorAction = {
      id: generateId(),
      directorId,
      type: actionType,
      data: actionData,
      result: actionResult,
      executedAt: new Date(),
      status: actionResult.success ? 'completed' : 'failed',
      requiresApproval: constraintCheck.requiresApproval
    };
    
    await this.recordDirectorAction(action);
    
    // Update power usage
    await this.updatePowerUsage(directorId, actionType);
    
    return {
      success: actionResult.success,
      reason: actionResult.reason,
      actionId: action.id
    };
  }
  
  /**
   * Activate emergency powers for the Director
   */
  async activateEmergencyPowers(
    directorId: string,
    justification: string,
    triggerEvent: EmergencyTrigger
  ): Promise<EmergencyActivationResult> {
    
    const director = await this.getDirector(directorId);
    
    // Validate emergency trigger
    const isValidTrigger = await this.validateEmergencyTrigger(triggerEvent);
    if (!isValidTrigger) {
      return {
        activated: false,
        reason: 'Invalid emergency trigger'
      };
    }
    
    // Check if emergency powers already active
    if (director.emergencyPowersActive) {
      return {
        activated: false,
        reason: 'Emergency powers already active'
      };
    }
    
    // Activate emergency powers
    director.emergencyPowersActive = true;
    director.emergencyActivatedAt = new Date();
    
    // Enhanced powers during emergency
    const emergencyPowers = this.getEmergencyPowers();
    director.powers.push(...emergencyPowers);
    
    await this.updateDirector(director);
    
    // Notify colloquium and all members
    await this.notifyEmergencyActivation(director.constitutionId, {
      directorId,
      justification,
      triggerEvent,
      activatedAt: new Date()
    });
    
    // Schedule automatic deactivation (max 30 days)
    const expirationDate = this.addDays(new Date(), 30);
    await this.scheduleEmergencyExpiration(directorId, expirationDate);
    
    return {
      activated: true,
      expirationDate,
      enhancedPowers: emergencyPowers.map(p => p.type)
    };
  }
  
  /**
   * Create new rule using director powers
   */
  private async createRule(
    director: Director,
    ruleData: NewRuleData
  ): Promise<ActionResult> {
    
    const ruleCost = this.calculateRuleCost(ruleData);
    
    // Check director credit
    if (director.creditBalance.available < ruleCost) {
      return {
        success: false,
        reason: 'Insufficient director credit'
      };
    }
    
    // Validate rule doesn't conflict with existing rules
    const validationResult = await this.validateNewRule(
      director.constitutionId,
      ruleData
    );
    
    if (!validationResult.valid) {
      return {
        success: false,
        reason: validationResult.reason
      };
    }
    
    // Create rule with pending status (requires colloquium approval)
    const newRule: ConstitutionalRule = {
      id: generateId(),
      title: ruleData.title,
      description: ruleData.description,
      category: ruleData.category,
      value: ruleData.initialValue,
      votingThreshold: ruleData.votingThreshold || 0.6,
      createdBy: director.userId,
      createdAt: new Date(),
      lastModified: new Date(),
      modificationHistory: [],
      dependencies: ruleData.dependencies || [],
      conflicts: []
    };
    
    // Save as pending rule
    await this.savePendingRule(newRule);
    
    // Deduct credit
    await this.deductDirectorCredit(director.userId, ruleCost);
    
    // Schedule colloquium vote
    await this.scheduleColloguiumVote(director.constitutionId, newRule.id);
    
    return {
      success: true,
      data: { ruleId: newRule.id, creditUsed: ruleCost }
    };
  }
  
  private getEmergencyPowers(): DirectorPower[] {
    return [
      {
        type: DirectorPowerType.EMERGENCY_OVERRIDE,
        description: 'Override any colloquium decision temporarily',
        constraints: ['time_limit:7_days', 'justification_required'],
        usageLimit: 3,
        usageCount: 0
      },
      {
        type: DirectorPowerType.MEMBER_SUSPENSION,
        description: 'Temporarily suspend members for safety/security',
        constraints: ['time_limit:14_days', 'appeal_allowed'],
        usageLimit: 5,
        usageCount: 0
      }
    ];
  }
}

interface EmergencyTrigger {
  type: 'security_threat' | 'system_failure' | 'external_crisis' | 'governance_deadlock';
  severity: 'low' | 'medium' | 'high' | 'critical';
  description: string;
  evidence: string[];
}

interface DirectorActionResult {
  success: boolean;
  reason?: string;
  actionId: string | null;
}

interface EmergencyActivationResult {
  activated: boolean;
  reason?: string;
  expirationDate?: Date;
  enhancedPowers?: DirectorPowerType[];
}
```

## Real-time Voting Interface

### Live Voting System
```typescript
class LiveVotingEngine {
  private socketService: SocketService;
  private activeVotes: Map<string, ActiveVote> = new Map();
  
  constructor(socketService: SocketService) {
    this.socketService = socketService;
  }
  
  /**
   * Initialize a live voting session
   */
  async startLiveVote(
    voteConfig: VoteConfiguration
  ): Promise<LiveVotingSession> {
    
    const session: LiveVotingSession = {
      id: generateId(),
      type: voteConfig.type,
      title: voteConfig.title,
      description: voteConfig.description,
      options: voteConfig.options,
      eligibleVoters: await this.getEligibleVoters(voteConfig),
      startTime: new Date(),
      endTime: this.addMinutes(new Date(), voteConfig.durationMinutes),
      requiredMajority: voteConfig.requiredMajority,
      status: 'active',
      votes: [],
      realTimeResults: {
        totalVotes: 0,
        totalWeight: 0,
        optionResults: voteConfig.options.map(option => ({
          optionId: option.id,
          voteCount: 0,
          totalWeight: 0,
          percentage: 0
        }))
      }
    };
    
    this.activeVotes.set(session.id, {
      session,
      participants: new Set()
    });
    
    await this.saveLiveVotingSession(session);
    
    // Create WebSocket room for real-time updates
    this.socketService.createRoom(`vote-${session.id}`);
    
    // Notify eligible voters
    await this.notifyEligibleVoters(session);
    
    // Schedule automatic closure
    setTimeout(() => {
      this.closeLiveVote(session.id);
    }, voteConfig.durationMinutes * 60 * 1000);
    
    return session;
  }
  
  /**
   * Process a live vote submission
   */
  async submitLiveVote(
    sessionId: string,
    voterId: string,
    optionId: string,
    reasoning?: string
  ): Promise<VoteSubmissionResult> {
    
    const activeVote = this.activeVotes.get(sessionId);
    if (!activeVote) {
      return { success: false, reason: 'Voting session not found or closed' };
    }
    
    const { session } = activeVote;
    
    // Validate voter eligibility
    const voter = session.eligibleVoters.find(v => v.userId === voterId);
    if (!voter) {
      return { success: false, reason: 'Not eligible to vote in this session' };
    }
    
    // Check if already voted
    const existingVote = session.votes.find(v => v.voterId === voterId);
    if (existingVote) {
      return { success: false, reason: 'Already voted in this session' };
    }
    
    // Validate option
    const option = session.options.find(o => o.id === optionId);
    if (!option) {
      return { success: false, reason: 'Invalid vote option' };
    }
    
    // Record vote
    const vote: LiveVote = {
      id: generateId(),
      sessionId,
      voterId,
      optionId,
      votingWeight: voter.votingWeight,
      reasoning,
      submittedAt: new Date()
    };
    
    session.votes.push(vote);
    activeVote.participants.add(voterId);
    
    // Update real-time results
    this.updateRealTimeResults(session);
    
    // Broadcast update to all participants
    this.socketService.emitToRoom(`vote-${sessionId}`, 'vote:update', {
      totalVotes: session.realTimeResults.totalVotes,
      results: session.realTimeResults.optionResults,
      newVoter: voter.userId === voterId ? undefined : 'anonymous' // Preserve anonymity option
    });
    
    await this.updateLiveVotingSession(session);
    
    return { success: true };
  }
  
  /**
   * Update real-time voting results
   */
  private updateRealTimeResults(session: LiveVotingSession): void {
    const results = session.realTimeResults;
    
    // Reset counts
    results.totalVotes = session.votes.length;
    results.totalWeight = session.votes.reduce((sum, vote) => sum + vote.votingWeight, 0);
    
    // Calculate per-option results
    results.optionResults.forEach(optionResult => {
      const optionVotes = session.votes.filter(vote => vote.optionId === optionResult.optionId);
      
      optionResult.voteCount = optionVotes.length;
      optionResult.totalWeight = optionVotes.reduce((sum, vote) => sum + vote.votingWeight, 0);
      optionResult.percentage = results.totalWeight > 0 ? 
        (optionResult.totalWeight / results.totalWeight) * 100 : 0;
    });
  }
  
  /**
   * Close a live voting session and calculate final results
   */
  async closeLiveVote(sessionId: string): Promise<VotingResult> {
    const activeVote = this.activeVotes.get(sessionId);
    if (!activeVote) {
      throw new Error('Voting session not found');
    }
    
    const { session } = activeVote;
    session.status = 'closed';
    session.actualEndTime = new Date();
    
    // Calculate final results
    const finalResults = this.calculateFinalResults(session);
    session.finalResults = finalResults;
    
    await this.updateLiveVotingSession(session);
    
    // Notify all participants of final results
    this.socketService.emitToRoom(`vote-${sessionId}`, 'vote:final', finalResults);
    
    // Clean up
    this.activeVotes.delete(sessionId);
    this.socketService.destroyRoom(`vote-${sessionId}`);
    
    return finalResults;
  }
  
  private calculateFinalResults(session: LiveVotingSession): VotingResult {
    const totalWeight = session.votes.reduce((sum, vote) => sum + vote.votingWeight, 0);
    const totalPossibleWeight = session.eligibleVoters.reduce((sum, voter) => 
      sum + voter.votingWeight, 0);
    
    // Check quorum (assuming 50% participation required)
    const participationRate = totalWeight / totalPossibleWeight;
    const quorumMet = participationRate >= 0.5;
    
    if (!quorumMet) {
      return {
        passed: false,
        reason: 'Quorum not met',
        participationRate,
        results: session.realTimeResults.optionResults
      };
    }
    
    // Determine winning option
    const sortedOptions = [...session.realTimeResults.optionResults]
      .sort((a, b) => b.totalWeight - a.totalWeight);
    
    const winningOption = sortedOptions[0];
    const winningPercentage = winningOption.totalWeight / totalWeight;
    
    const passed = winningPercentage >= session.requiredMajority;
    
    return {
      passed,
      winningOption: winningOption.optionId,
      winningPercentage,
      participationRate,
      results: session.realTimeResults.optionResults
    };
  }
}

interface LiveVotingSession {
  id: string;
  type: string;
  title: string;
  description: string;
  options: VoteOption[];
  eligibleVoters: EligibleVoter[];
  startTime: Date;
  endTime: Date;
  actualEndTime?: Date;
  requiredMajority: number;
  status: 'active' | 'closed';
  votes: LiveVote[];
  realTimeResults: RealTimeResults;
  finalResults?: VotingResult;
}

interface ActiveVote {
  session: LiveVotingSession;
  participants: Set<string>;
}

interface LiveVote {
  id: string;
  sessionId: string;
  voterId: string;
  optionId: string;
  votingWeight: number;
  reasoning?: string;
  submittedAt: Date;
}
```

## Database Schema

### Governance Tables
```sql
-- Constitutions
CREATE TABLE constitutions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  version INTEGER NOT NULL,
  scope VARCHAR(20) NOT NULL,
  scope_id UUID,
  founding_principles JSONB NOT NULL,
  ratification_date TIMESTAMP NOT NULL,
  last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  next_review_date TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Constitutional rules
CREATE TABLE constitutional_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  constitution_id UUID REFERENCES constitutions(id) ON DELETE CASCADE,
  title VARCHAR(500) NOT NULL,
  description TEXT NOT NULL,
  category VARCHAR(50) NOT NULL,
  value INTEGER CHECK (value >= 0 AND value <= 100),
  voting_threshold DECIMAL(3,2) CHECK (voting_threshold >= 0.5 AND voting_threshold <= 0.95),
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  dependencies JSONB DEFAULT '[]',
  conflicts JSONB DEFAULT '[]'
);

-- Rule votes
CREATE TABLE rule_votes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rule_id UUID REFERENCES constitutional_rules(id) ON DELETE CASCADE,
  voter_id UUID REFERENCES users(id) ON DELETE CASCADE,
  vote_type VARCHAR(20) NOT NULL CHECK (vote_type IN ('strengthen', 'weaken', 'remove')),
  voting_weight DECIMAL(8,2) NOT NULL,
  cycle INTEGER NOT NULL,
  reasoning TEXT,
  cast_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(rule_id, voter_id, cycle)
);

-- Colloquium
CREATE TABLE colloquia (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  constitution_id UUID REFERENCES constitutions(id) ON DELETE CASCADE,
  current_cycle INTEGER NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE colloquium_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  colloquium_id UUID REFERENCES colloquia(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  representing_subject VARCHAR(200) NOT NULL,
  voting_weight DECIMAL(8,2) NOT NULL,
  joined_cycle INTEGER NOT NULL,
  status VARCHAR(20) DEFAULT 'active',
  attendance_rate DECIMAL(5,2) DEFAULT 100.00,
  
  UNIQUE(colloquium_id, subject_id)
);

-- Colloquium sessions
CREATE TABLE colloquium_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  colloquium_id UUID REFERENCES colloquia(id) ON DELETE CASCADE,
  session_number INTEGER NOT NULL,
  cycle INTEGER NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP,
  status VARCHAR(20) DEFAULT 'scheduled',
  agenda JSONB NOT NULL,
  attendees JSONB DEFAULT '[]',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Colloquium decisions
CREATE TABLE colloquium_decisions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES colloquium_sessions(id) ON DELETE CASCADE,
  agenda_item_id UUID NOT NULL,
  decision_type VARCHAR(50) NOT NULL,
  passed BOOLEAN NOT NULL,
  support_weight DECIMAL(10,2) NOT NULL,
  oppose_weight DECIMAL(10,2) NOT NULL,
  abstain_weight DECIMAL(10,2) NOT NULL,
  required_majority DECIMAL(3,2) NOT NULL,
  actual_majority DECIMAL(5,2) NOT NULL,
  votes JSONB NOT NULL,
  decided_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  effective_date TIMESTAMP NOT NULL
);

-- Directors
CREATE TABLE directors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  constitution_id UUID REFERENCES constitutions(id) ON DELETE CASCADE,
  elected_cycle INTEGER NOT NULL,
  term_start_date TIMESTAMP NOT NULL,
  term_end_date TIMESTAMP NOT NULL,
  emergency_powers_active BOOLEAN DEFAULT FALSE,
  emergency_activated_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(constitution_id, elected_cycle)
);

-- Director actions
CREATE TABLE director_actions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  director_id UUID REFERENCES directors(id) ON DELETE CASCADE,
  action_type VARCHAR(50) NOT NULL,
  action_data JSONB NOT NULL,
  result JSONB,
  executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  status VARCHAR(20) DEFAULT 'completed',
  requires_approval BOOLEAN DEFAULT FALSE,
  approved_at TIMESTAMP,
  approved_by UUID REFERENCES users(id)
);

-- Live voting sessions
CREATE TABLE live_voting_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type VARCHAR(50) NOT NULL,
  title VARCHAR(500) NOT NULL,
  description TEXT,
  options JSONB NOT NULL,
  eligible_voters JSONB NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP NOT NULL,
  actual_end_time TIMESTAMP,
  required_majority DECIMAL(3,2) NOT NULL,
  status VARCHAR(20) DEFAULT 'active',
  real_time_results JSONB,
  final_results JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE live_votes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID REFERENCES live_voting_sessions(id) ON DELETE CASCADE,
  voter_id UUID REFERENCES users(id) ON DELETE CASCADE,
  option_id UUID NOT NULL,
  voting_weight DECIMAL(8,2) NOT NULL,
  reasoning TEXT,
  submitted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(session_id, voter_id)
);

-- Indexes
CREATE INDEX idx_constitutional_rules_constitution ON constitutional_rules(constitution_id);
CREATE INDEX idx_rule_votes_rule_cycle ON rule_votes(rule_id, cycle);
CREATE INDEX idx_colloquium_members_colloquium ON colloquium_members(colloquium_id);
CREATE INDEX idx_colloquium_decisions_session ON colloquium_decisions(session_id);
CREATE INDEX idx_director_actions_director ON director_actions(director_id);
CREATE INDEX idx_live_votes_session ON live_votes(session_id);
```


This governance and voting system provides the sophisticated meritocratic framework needed for Peer Academy, balancing weighted voting based on merit, evolutionary rule systems, constrained director powers, and transparent decision-making processes.

