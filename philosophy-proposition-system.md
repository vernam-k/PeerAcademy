# Peer Academy - Philosophy & Proposition Management System

## Overview

The philosophy and proposition management system is the intellectual core of Peer Academy, designed to build a shared worldview through systematic knowledge integration. This system captures insights from weekly presentations, organizes them into coherent philosophical branches, validates logical consistency, and evolves a living body of knowledge through weighted community consensus.

## Philosophy Framework Architecture

### Philosophical Structure
```typescript
interface PhilosophySystem {
  id: string;
  academyId: string;
  branches: PhilosophyBranch[];
  globalConsistencyRules: ConsistencyRule[];
  logicEngine: LogicValidationEngine;
  consensusThresholds: ConsensusThresholds;
  evolutionHistory: PhilosophyEvolution[];
}

interface PhilosophyBranch {
  id: string;
  name: string;
  description: string;
  foundationalConcepts: string[];
  subjects: string[]; // subject IDs that contribute to this branch
  propositions: Proposition[];
  logicRules: BranchLogicRules;
  curator: string; // user ID of branch curator
  createdAt: Date;
  lastUpdated: Date;
}

interface Proposition {
  id: string;
  branchId: string;
  subjectId: string; // originating subject
  presentationId?: string; // source presentation if applicable
  
  // Content
  title: string;
  statement: string;
  evidence: Evidence[];
  reasoning: string;
  scope: PropositionScope;
  
  // Validation and relationships
  logicalForm: LogicalForm;
  dependencies: string[]; // other proposition IDs this depends on
  supports: string[]; // propositions this supports
  conflicts: string[]; // propositions this conflicts with
  
  // Community consensus
  value: number; // 0-100, current strength/acceptance
  votes: PropositionVote[];
  reviews: PropositionReview[];
  
  // Lifecycle
  status: PropositionStatus;
  createdBy: string;
  createdAt: Date;
  lastModified: Date;
  lastReview: Date;
}

enum PropositionStatus {
  DRAFT = 'draft',
  UNDER_REVIEW = 'under_review', 
  ACTIVE = 'active',
  DISPUTED = 'disputed',
  DEPRECATED = 'deprecated',
  DOGMA = 'dogma' // Value > 90, very hard to challenge
}

enum PropositionScope {
  UNIVERSAL = 'universal', // Applies to all contexts
  CONTEXTUAL = 'contextual', // Applies to specific contexts
  CONDITIONAL = 'conditional', // Applies under certain conditions
  EXPERIMENTAL = 'experimental' // Tentative, needs more evidence
}

interface Evidence {
  type: 'empirical' | 'logical' | 'testimonial' | 'authoritative';
  description: string;
  source: string;
  confidence: number; // 0-1
  dateAdded: Date;
  addedBy: string;
}

interface LogicalForm {
  premises: string[];
  conclusion: string;
  inferenceRule: string;
  formalRepresentation?: string; // For advanced logical analysis
}
```

### Proposition Creation Engine
```typescript
class PropositionEngine {
  private logicValidator: LogicValidationEngine;
  private consistencyChecker: ConsistencyChecker;
  
  /**
   * Extract and create propositions from presentations
   */
  async extractPropositionsFromPresentation(
    presentationId: string,
    extractorId: string
  ): Promise<PropositionExtractionResult> {
    
    const presentation = await this.getPresentation(presentationId);
    const subject = await this.getSubject(presentation.subjectId);
    
    // Validate extractor permissions
    await this.validateExtractionPermissions(extractorId, presentation.subjectId);
    
    // Use AI/NLP to identify potential propositions from content
    const candidatePropositions = await this.identifyPropositions(presentation);
    
    // Allow human refinement of extracted propositions
    const refinedPropositions = await this.refinePropositions(
      candidatePropositions,
      extractorId
    );
    
    // Create proposition records
    const createdPropositions: Proposition[] = [];
    
    for (const candidate of refinedPropositions) {
      const proposition = await this.createProposition({
        branchId: await this.determineBranch(candidate, subject),
        subjectId: presentation.subjectId,
        presentationId,
        title: candidate.title,
        statement: candidate.statement,
        evidence: candidate.evidence,
        reasoning: candidate.reasoning,
        scope: candidate.scope,
        logicalForm: candidate.logicalForm,
        createdBy: extractorId
      });
      
      createdPropositions.push(proposition);
    }
    
    return {
      presentationId,
      extractedCount: createdPropositions.length,
      propositions: createdPropositions,
      requiresReview: createdPropositions.some(p => p.status === PropositionStatus.UNDER_REVIEW)
    };
  }
  
  /**
   * Create a new proposition with full validation
   */
  async createProposition(data: CreatePropositionData): Promise<Proposition> {
    // Validate logical form
    const logicValidation = await this.logicValidator.validate(data.logicalForm);
    if (!logicValidation.valid) {
      throw new Error(`Invalid logical form: ${logicValidation.errors.join(', ')}`);
    }
    
    // Check for conflicts with existing propositions
    const conflictCheck = await this.consistencyChecker.checkConflicts(
      data.branchId,
      data.statement,
      data.logicalForm
    );
    
    const proposition: Proposition = {
      id: generateId(),
      branchId: data.branchId,
      subjectId: data.subjectId,
      presentationId: data.presentationId,
      title: data.title,
      statement: data.statement,
      evidence: data.evidence,
      reasoning: data.reasoning,
      scope: data.scope,
      logicalForm: data.logicalForm,
      dependencies: data.dependencies || [],
      supports: [],
      conflicts: conflictCheck.conflicts,
      value: 50, // Start at neutral value
      votes: [],
      reviews: [],
      status: conflictCheck.conflicts.length > 0 ? 
        PropositionStatus.DISPUTED : PropositionStatus.ACTIVE,
      createdBy: data.createdBy,
      createdAt: new Date(),
      lastModified: new Date(),
      lastReview: new Date()
    };
    
    await this.saveProposition(proposition);
    
    // Update related propositions
    if (conflictCheck.conflicts.length > 0) {
      await this.flagConflictingPropositions(conflictCheck.conflicts, proposition.id);
    }
    
    // Trigger consistency review if needed
    if (proposition.status === PropositionStatus.DISPUTED) {
      await this.scheduleConsistencyReview(proposition.branchId);
    }
    
    return proposition;
  }
  
  /**
   * Use AI to identify potential propositions from presentation content
   */
  private async identifyPropositions(presentation: Presentation): Promise<CandidateProposition[]> {
    // This would integrate with an AI service to analyze content
    // For now, showing the interface structure
    
    const content = await this.extractTextContent(presentation);
    
    // Analyze content for claims, arguments, and evidence
    const analysis = await this.aiAnalysisService.analyzePhilosophicalContent(content);
    
    return analysis.candidatePropositions.map(candidate => ({
      title: candidate.title,
      statement: candidate.statement,
      evidence: candidate.evidence.map(e => ({
        type: e.type as any,
        description: e.description,
        source: presentation.title,
        confidence: e.confidence,
        dateAdded: new Date(),
        addedBy: presentation.presenterId
      })),
      reasoning: candidate.reasoning,
      scope: this.determineScope(candidate),
      logicalForm: candidate.logicalForm,
      confidence: candidate.confidence
    }));
  }
}

interface CandidateProposition {
  title: string;
  statement: string;
  evidence: Evidence[];
  reasoning: string;
  scope: PropositionScope;
  logicalForm: LogicalForm;
  confidence: number;
}
```

## Logic Validation System

### Consistency Checking Engine
```typescript
class ConsistencyChecker {
  private reasoningEngine: ReasoningEngine;
  
  /**
   * Check for logical conflicts between propositions
   */
  async checkConflicts(
    branchId: string,
    newStatement: string,
    newLogicalForm: LogicalForm
  ): Promise<ConflictCheckResult> {
    
    const branch = await this.getBranch(branchId);
    const activePropositions = branch.propositions.filter(p => 
      p.status === PropositionStatus.ACTIVE || p.status === PropositionStatus.DOGMA
    );
    
    const conflicts: string[] = [];
    const potentialConflicts: string[] = [];
    
    for (const existing of activePropositions) {
      const conflictType = await this.detectConflict(newLogicalForm, existing.logicalForm);
      
      switch (conflictType) {
        case ConflictType.DIRECT_CONTRADICTION:
          conflicts.push(existing.id);
          break;
        case ConflictType.POTENTIAL_INCONSISTENCY:
          potentialConflicts.push(existing.id);
          break;
      }
    }
    
    return {
      hasConflicts: conflicts.length > 0,
      conflicts,
      potentialConflicts,
      analysis: await this.generateConflictAnalysis(conflicts, potentialConflicts)
    };
  }
  
  /**
   * Detect type of conflict between two logical forms
   */
  private async detectConflict(
    form1: LogicalForm,
    form2: LogicalForm
  ): Promise<ConflictType> {
    
    // Direct contradiction: if conclusion of one negates conclusion of other
    if (this.isDirectNegation(form1.conclusion, form2.conclusion)) {
      return ConflictType.DIRECT_CONTRADICTION;
    }
    
    // Check for premise conflicts
    const premiseConflicts = this.checkPremiseConflicts(form1.premises, form2.premises);
    if (premiseConflicts.length > 0) {
      return ConflictType.PREMISE_CONFLICT;
    }
    
    // Use reasoning engine for deeper analysis
    const reasoningResult = await this.reasoningEngine.checkConsistency([form1, form2]);
    if (!reasoningResult.consistent) {
      return ConflictType.LOGICAL_INCONSISTENCY;
    }
    
    // Check for potential future conflicts
    const dependencyAnalysis = await this.analyzeDependencies(form1, form2);
    if (dependencyAnalysis.mayConflict) {
      return ConflictType.POTENTIAL_INCONSISTENCY;
    }
    
    return ConflictType.NO_CONFLICT;
  }
  
  /**
   * Comprehensive branch consistency check
   */
  async checkBranchConsistency(branchId: string): Promise<ConsistencyReport> {
    const branch = await this.getBranch(branchId);
    const activePropositions = branch.propositions.filter(p => 
      p.status === PropositionStatus.ACTIVE || p.status === PropositionStatus.DOGMA
    );
    
    const conflicts: ConflictGroup[] = [];
    const circularDependencies: string[][] = [];
    const logicalGaps: LogicalGap[] = [];
    
    // Pairwise conflict detection
    for (let i = 0; i < activePropositions.length; i++) {
      for (let j = i + 1; j < activePropositions.length; j++) {
        const conflictType = await this.detectConflict(
          activePropositions[i].logicalForm,
          activePropositions[j].logicalForm
        );
        
        if (conflictType !== ConflictType.NO_CONFLICT) {
          conflicts.push({
            propositions: [activePropositions[i].id, activePropositions[j].id],
            type: conflictType,
            severity: this.calculateConflictSeverity(
              activePropositions[i],
              activePropositions[j],
              conflictType
            )
          });
        }
      }
    }
    
    // Check for circular dependencies
    circularDependencies.push(...this.findCircularDependencies(activePropositions));
    
    // Identify logical gaps
    logicalGaps.push(...await this.identifyLogicalGaps(activePropositions));
    
    return {
      branchId,
      overallConsistency: conflicts.length === 0 && circularDependencies.length === 0,
      conflictCount: conflicts.length,
      conflicts,
      circularDependencies,
      logicalGaps,
      recommendations: await this.generateConsistencyRecommendations(
        conflicts,
        circularDependencies,
        logicalGaps
      )
    };
  }
}

enum ConflictType {
  NO_CONFLICT = 'no_conflict',
  DIRECT_CONTRADICTION = 'direct_contradiction',
  PREMISE_CONFLICT = 'premise_conflict',
  LOGICAL_INCONSISTENCY = 'logical_inconsistency',
  POTENTIAL_INCONSISTENCY = 'potential_inconsistency'
}

interface ConflictGroup {
  propositions: string[];
  type: ConflictType;
  severity: 'low' | 'medium' | 'high' | 'critical';
}

interface LogicalGap {
  description: string;
  missingPremises: string[];
  affectedPropositions: string[];
  suggestedResolution: string;
}
```

## Proposition Voting and Evolution

### Weighted Proposition Voting
```typescript
class PropositionVotingEngine {
  /**
   * Process votes on a proposition and update its value
   */
  async processPropositionVotes(
    propositionId: string,
    cycle: number
  ): Promise<PropositionEvolutionResult> {
    
    const proposition = await this.getProposition(propositionId);
    const votes = await this.getPropositionVotes(propositionId, cycle);
    
    if (votes.length === 0) {
      return { propositionId, valueChanged: false, newValue: proposition.value };
    }
    
    // Calculate weighted vote impact
    const voteImpact = this.calculateVoteImpact(votes, proposition);
    
    // Apply value change with constraints
    const newValue = this.applyValueChange(proposition.value, voteImpact);
    
    // Determine new status based on value
    const newStatus = this.determineStatus(newValue, proposition.status);
    
    // Update proposition
    await this.updateProposition(propositionId, {
      value: newValue,
      status: newStatus,
      lastModified: new Date()
    });
    
    // Check for status transitions
    const statusChanged = newStatus !== proposition.status;
    if (statusChanged) {
      await this.handleStatusTransition(propositionId, proposition.status, newStatus);
    }
    
    return {
      propositionId,
      valueChanged: Math.abs(newValue - proposition.value) >= 1,
      oldValue: proposition.value,
      newValue,
      statusChanged,
      oldStatus: proposition.status,
      newStatus,
      voteDetails: {
        totalVotes: votes.length,
        totalWeight: votes.reduce((sum, v) => sum + v.votingWeight, 0),
        supportWeight: votes.filter(v => v.voteType === 'support')
          .reduce((sum, v) => sum + v.votingWeight, 0),
        opposeWeight: votes.filter(v => v.voteType === 'oppose')
          .reduce((sum, v) => sum + v.votingWeight, 0)
      }
    };
  }
  
  /**
   * Calculate the impact of votes on proposition value
   */
  private calculateVoteImpact(
    votes: PropositionVote[],
    proposition: Proposition
  ): number {
    
    let supportWeight = 0;
    let opposeWeight = 0;
    let strengthenWeight = 0;
    let weakenWeight = 0;
    
    votes.forEach(vote => {
      switch (vote.voteType) {
        case 'support':
          supportWeight += vote.votingWeight;
          break;
        case 'oppose':
          opposeWeight += vote.votingWeight;
          break;
        case 'strengthen':
          strengthenWeight += vote.votingWeight;
          break;
        case 'weaken':
          weakenWeight += vote.votingWeight;
          break;
      }
    });
    
    const totalWeight = supportWeight + opposeWeight + strengthenWeight + weakenWeight;
    
    if (totalWeight === 0) return 0;
    
    // Net support impact
    const netSupport = (supportWeight - opposeWeight) / totalWeight;
    
    // Net strengthening impact
    const netStrengthen = (strengthenWeight - weakenWeight) / totalWeight;
    
    // Combine impacts with resistance based on current value
    const resistance = this.calculateResistance(proposition.value);
    const baseImpact = (netSupport + netStrengthen) * 10; // Base 10-point scale
    
    return baseImpact * (1 - resistance);
  }
  
  /**
   * Calculate resistance to change based on current value
   */
  private calculateResistance(currentValue: number): number {
    // Higher values (approaching dogma) resist change more
    // Lower values (approaching elimination) resist change more
    
    if (currentValue >= 90) {
      // Dogma territory - very high resistance
      return 0.8;
    } else if (currentValue <= 10) {
      // Near elimination - high resistance to prevent flip-flopping
      return 0.6;
    } else if (currentValue >= 70 || currentValue <= 30) {
      // Moderate extremes - medium resistance
      return 0.4;
    } else {
      // Middle ground - low resistance
      return 0.2;
    }
  }
  
  /**
   * Apply value change with bounds and stability measures
   */
  private applyValueChange(currentValue: number, impact: number): number {
    // Apply change with diminishing returns at extremes
    let newValue = currentValue + impact;
    
    // Bounds checking
    newValue = Math.max(0, Math.min(100, newValue));
    
    // Stability measure: prevent rapid oscillation
    const maxChangePerCycle = 15;
    if (Math.abs(impact) > maxChangePerCycle) {
      const sign = impact > 0 ? 1 : -1;
      newValue = currentValue + (sign * maxChangePerCycle);
    }
    
    return Math.round(newValue);
  }
  
  /**
   * Determine proposition status based on value
   */
  private determineStatus(value: number, currentStatus: PropositionStatus): PropositionStatus {
    if (value >= 95) {
      return PropositionStatus.DOGMA;
    } else if (value <= 5) {
      return PropositionStatus.DEPRECATED;
    } else if (value >= 70) {
      return PropositionStatus.ACTIVE;
    } else if (value <= 30) {
      return currentStatus === PropositionStatus.DISPUTED ? 
        PropositionStatus.DISPUTED : PropositionStatus.ACTIVE;
    } else {
      return PropositionStatus.ACTIVE;
    }
  }
  
  /**
   * Handle status transitions and their consequences
   */
  private async handleStatusTransition(
    propositionId: string,
    oldStatus: PropositionStatus,
    newStatus: PropositionStatus
  ): Promise<void> {
    
    const proposition = await this.getProposition(propositionId);
    
    switch (newStatus) {
      case PropositionStatus.DOGMA:
        // Becoming dogma - notify academy, make very hard to challenge
        await this.notifyDogmaTransition(proposition);
        await this.updateVotingThreshold(propositionId, 0.9); // 90% required to challenge
        break;
        
      case PropositionStatus.DEPRECATED:
        // Being deprecated - archive and remove from active philosophy
        await this.archiveProposition(proposition);
        await this.updateRelatedPropositions(propositionId, 'remove_dependency');
        break;
        
      case PropositionStatus.DISPUTED:
        // Becoming disputed - trigger review process
        await this.scheduleDisputeReview(propositionId);
        break;
        
      case PropositionStatus.ACTIVE:
        if (oldStatus === PropositionStatus.DISPUTED) {
          // Resolved dispute
          await this.notifyDisputeResolution(proposition);
        }
        break;
    }
  }
}

interface PropositionVote {
  id: string;
  propositionId: string;
  voterId: string;
  voteType: 'support' | 'oppose' | 'strengthen' | 'weaken';
  votingWeight: number;
  reasoning: string;
  evidence?: Evidence[];
  cycle: number;
  castAt: Date;
}
```

## Philosophy Integration Engine

### Knowledge Synthesis System
```typescript
class PhilosophyIntegrationEngine {
  /**
   * Generate integrated philosophical framework from branch propositions
   */
  async synthesizePhilosophicalFramework(
    academyId: string
  ): Promise<PhilosophicalFramework> {
    
    const branches = await this.getPhilosophyBranches(academyId);
    const activePropositions = this.getActivePropositions(branches);
    
    // Group propositions by themes and relationships
    const thematicGroups = await this.identifyThematicGroups(activePropositions);
    
    // Build hierarchical structure
    const hierarchy = await this.buildConceptualHierarchy(thematicGroups);
    
    // Generate synthesis document
    const synthesis = await this.generateSynthesis(hierarchy, branches);
    
    // Identify knowledge gaps
    const gaps = await this.identifyKnowledgeGaps(hierarchy, activePropositions);
    
    return {
      academyId,
      version: await this.getNextVersionNumber(academyId),
      generatedAt: new Date(),
      branches: branches.map(b => ({
        branchId: b.id,
        name: b.name,
        propositionCount: b.propositions.filter(p => p.status === PropositionStatus.ACTIVE).length,
        dogmaCount: b.propositions.filter(p => p.status === PropositionStatus.DOGMA).length,
        consistency: this.calculateBranchConsistency(b)
      })),
      thematicGroups,
      conceptualHierarchy: hierarchy,
      synthesis,
      knowledgeGaps: gaps,
      recommendations: await this.generateFrameworkRecommendations(hierarchy, gaps)
    };
  }
  
  /**
   * Identify thematic groups across branches
   */
  private async identifyThematicGroups(
    propositions: Proposition[]
  ): Promise<ThematicGroup[]> {
    
    const groups: ThematicGroup[] = [];
    
    // Use semantic analysis to cluster related propositions
    const semanticClusters = await this.semanticAnalysisService.clusterPropositions(
      propositions.map(p => ({
        id: p.id,
        text: `${p.title} ${p.statement} ${p.reasoning}`,
        branchId: p.branchId
      }))
    );
    
    for (const cluster of semanticClusters) {
      const clusterPropositions = propositions.filter(p => 
        cluster.propositionIds.includes(p.id)
      );
      
      groups.push({
        id: generateId(),
        theme: cluster.theme,
        description: cluster.description,
        propositions: clusterPropositions.map(p => p.id),
        crossBranch: this.hasCrossBranchPropositions(clusterPropositions),
        coherenceScore: cluster.coherenceScore,
        keywords: cluster.keywords
      });
    }
    
    return groups;
  }
  
  /**
   * Build conceptual hierarchy from thematic groups
   */
  private async buildConceptualHierarchy(
    groups: ThematicGroup[]
  ): Promise<ConceptualNode[]> {
    
    // Create nodes for each thematic group
    const nodes: ConceptualNode[] = groups.map(group => ({
      id: group.id,
      concept: group.theme,
      description: group.description,
      level: 0, // Will be updated
      children: [],
      parent: null,
      propositions: group.propositions,
      strength: group.coherenceScore
    }));
    
    // Build hierarchical relationships
    const hierarchy = await this.buildHierarchicalRelations(nodes);
    
    return hierarchy;
  }
  
  /**
   * Generate synthesis document from philosophical framework
   */
  private async generateSynthesis(
    hierarchy: ConceptualNode[],
    branches: PhilosophyBranch[]
  ): Promise<PhilosophySynthesis> {
    
    // Generate executive summary
    const executiveSummary = await this.generateExecutiveSummary(hierarchy);
    
    // Create detailed synthesis for each major concept
    const conceptualSyntheses = await Promise.all(
      hierarchy.filter(node => node.level === 0) // Top-level concepts
        .map(node => this.synthesizeConcept(node, branches))
    );
    
    // Identify core principles
    const corePrinciples = await this.extractCorePrinciples(hierarchy);
    
    // Generate worldview statement
    const worldviewStatement = await this.generateWorldviewStatement(
      corePrinciples,
      conceptualSyntheses
    );
    
    return {
      executiveSummary,
      worldviewStatement,
      corePrinciples,
      conceptualSyntheses,
      methodologyNotes: await this.generateMethodologyNotes(),
      confidence: this.calculateOverallConfidence(hierarchy)
    };
  }
}

interface ThematicGroup {
  id: string;
  theme: string;
  description: string;
  propositions: string[];
  crossBranch: boolean;
  coherenceScore: number;
  keywords: string[];
}

interface ConceptualNode {
  id: string;
  concept: string;
  description: string;
  level: number;
  children: string[];
  parent: string | null;
  propositions: string[];
  strength: number;
}

interface PhilosophicalFramework {
  academyId: string;
  version: number;
  generatedAt: Date;
  branches: BranchSummary[];
  thematicGroups: ThematicGroup[];
  conceptualHierarchy: ConceptualNode[];
  synthesis: PhilosophySynthesis;
  knowledgeGaps: KnowledgeGap[];
  recommendations: string[];
}
```

## Advanced Search and Discovery

### Philosophical Query Engine
```typescript
class PhilosophicalQueryEngine {
  private searchIndex: SearchIndex;
  private semanticEngine: SemanticSearchEngine;
  
  /**
   * Advanced search across philosophical content
   */
  async queryPhilosophy(
    academyId: string,
    query: PhilosophicalQuery
  ): Promise<QueryResults> {
    
    const results: QueryResults = {
      query,
      totalResults: 0,
      propositions: [],
      concepts: [],
      relationships: [],
      suggestions: []
    };
    
    // Text-based search
    if (query.text) {
      const textResults = await this.searchIndex.search(query.text, {
        branches: query.branches,
        status: query.status,
        minValue: query.minValue,
        maxValue: query.maxValue
      });
      
      results.propositions.push(...textResults.propositions);
    }
    
    // Semantic search
    if (query.semantic) {
      const semanticResults = await this.semanticEngine.search(
        query.semantic,
        query.branches
      );
      
      results.propositions.push(...semanticResults);
    }
    
    // Logical structure search
    if (query.logicalForm) {
      const logicalResults = await this.searchByLogicalForm(query.logicalForm);
      results.propositions.push(...logicalResults);
    }
    
    // Relationship search
    if (query.relationships) {
      const relationshipResults = await this.searchByRelationships(
        query.relationships
      );
      
      results.relationships.push(...relationshipResults);
    }
    
    // Remove duplicates and apply ranking
    results.propositions = this.deduplicateAndRank(results.propositions, query);
    results.totalResults = results.propositions.length;
    
    // Generate related concepts
    results.concepts = await this.findRelatedConcepts(results.propositions);
    
    // Generate search suggestions
    results.suggestions = await this.generateSearchSuggestions(query, results);
    
    return results;
  }
  
  /**
   * Find propositions that explore philosophical tensions or disagreements
   */
  async findPhilosophicalTensions(
    branchId?: string
  ): Promise<PhilosophicalTension[]> {
    
    const scope = branchId ? [branchId] : await this.getAllBranchIds();
    const tensions: PhilosophicalTension[] = [];
    
    for (const branch of scope) {
      const conflicts = await this.findActiveConflicts(branch);
      
      for (const conflict of conflicts) {
        const tension = await this.analyzeTension(conflict);
        tensions.push(tension);
      }
    }
    
    // Rank tensions by philosophical significance
    return tensions.sort((a, b) => b.significance - a.significance);
  }
  
  /**
   * Generate philosophical research suggestions
   */
  async generateResearchSuggestions(
    userId: string,
    subjectId: string
  ): Promise<ResearchSuggestion[]> {
    
    const userInterests = await this.getUserPhilosophicalInterests(userId);
    const subjectFocus = await this.getSubjectPhilosophicalFocus(subjectId);
    const knowledgeGaps = await this.getKnowledgeGaps();
    
    const suggestions: ResearchSuggestion[] = [];
    
    // Suggest filling knowledge gaps
    for (const gap of knowledgeGaps) {
      if (this.matchesInterests(gap, userInterests, subjectFocus)) {
        suggestions.push({
          type: 'knowledge_gap',
          title: `Explore ${gap.topic}`,
          description: gap.description,
          difficulty: gap.difficulty,
          estimatedEffort: gap.estimatedEffort,
          relatedPropositions: gap.relatedPropositions,
          potentialImpact: gap.impact
        });
      }
    }
    
    // Suggest challenging existing propositions
    const challengeablePropositions = await this.findChallengeablePropositions(
      userInterests,
      subjectFocus
    );
    
    for (const prop of challengeablePropositions) {
      suggestions.push({
        type: 'challenge',
        title: `Challenge: ${prop.title}`,
        description: `This proposition has moderate support but may benefit from additional scrutiny`,
        difficulty: this.calculateChallengeDifficulty(prop),
        relatedPropositions: [prop.id],
        potentialImpact: this.calculateChallengeImpact(prop)
      });
    }
    
    return suggestions.sort((a, b) => b.potentialImpact - a.potentialImpact);
  }
}

interface PhilosophicalQuery {
  text?: string;
  semantic?: string;
  logicalForm?: Partial<LogicalForm>;
  branches?: string[];
  status?: PropositionStatus[];
  minValue?: number;
  maxValue?: number;
  relationships?: RelationshipQuery[];
}

interface PhilosophicalTension {
  id: string;
  title: string;
  description: string;
  conflictingPropositions: string[];
  significance: number;
  resolutionSuggestions: string[];
}

interface ResearchSuggestion {
  type: 'knowledge_gap' | 'challenge' | 'synthesis' | 'exploration';
  title: string;
  description: string;
  difficulty: 'beginner' | 'intermediate' | 'advanced' | 'expert';
  estimatedEffort?: string;
  relatedPropositions: string[];
  potentialImpact: number;
}
```

## Database Schema

### Philosophy Tables
```sql
-- Philosophy branches
CREATE TABLE philosophy_branches (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  academy_id UUID REFERENCES academies(id) ON DELETE CASCADE,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  foundational_concepts JSONB DEFAULT '[]',
  subjects JSONB NOT NULL, -- array of subject IDs
  curator_id UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Propositions
CREATE TABLE propositions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  branch_id UUID REFERENCES philosophy_branches(id) ON DELETE CASCADE,
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  presentation_id UUID REFERENCES presentations(id) ON DELETE SET NULL,
  
  title VARCHAR(500) NOT NULL,
  statement TEXT NOT NULL,
  evidence JSONB DEFAULT '[]',
  reasoning TEXT NOT NULL,
  scope VARCHAR(20) DEFAULT 'contextual',
  
  logical_form JSONB NOT NULL,
  dependencies JSONB DEFAULT '[]',
  supports JSONB DEFAULT '[]',
  conflicts JSONB DEFAULT '[]',
  
  value INTEGER CHECK (value >= 0 AND value <= 100) DEFAULT 50,
  status VARCHAR(20) DEFAULT 'active',
  
  created_by UUID REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_review TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Proposition votes
CREATE TABLE proposition_votes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  proposition_id UUID REFERENCES propositions(id) ON DELETE CASCADE,
  voter_id UUID REFERENCES users(id) ON DELETE CASCADE,
  vote_type VARCHAR(20) NOT NULL CHECK (vote_type IN ('support', 'oppose', 'strengthen', 'weaken')),
  voting_weight DECIMAL(8,2) NOT NULL,
  reasoning TEXT NOT NULL,
  evidence JSONB DEFAULT '[]',
  cycle INTEGER NOT NULL,
  cast_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(proposition_id, voter_id, cycle)
);

-- Proposition reviews
CREATE TABLE proposition_reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  proposition_id UUID REFERENCES propositions(id) ON DELETE CASCADE,
  reviewer_id UUID REFERENCES users(id) ON DELETE CASCADE,
  review_type VARCHAR(20) NOT NULL, -- 'consistency', 'evidence', 'logic', 'significance'
  rating INTEGER CHECK (rating >= 1 AND rating <= 10),
  feedback TEXT NOT NULL,
  recommendations JSONB DEFAULT '[]',
  approved BOOLEAN,
  reviewed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Consistency checks
CREATE TABLE consistency_checks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  branch_id UUID REFERENCES philosophy_branches(id) ON DELETE CASCADE,
  check_type VARCHAR(50) NOT NULL,
  propositions_involved JSONB NOT NULL,
  result VARCHAR(20) NOT NULL, -- 'consistent', 'conflict', 'potential_conflict'
  details JSONB,
  severity VARCHAR(10), -- 'low', 'medium', 'high', 'critical'
  resolved_at TIMESTAMP,
  resolved_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Philosophical frameworks (synthesis results)
CREATE TABLE philosophical_frameworks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  academy_id UUID REFERENCES academies(id) ON DELETE CASCADE,
  version INTEGER NOT NULL,
  framework_data JSONB NOT NULL,
  generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  generated_by VARCHAR(50) DEFAULT 'system',
  
  UNIQUE(academy_id, version)
);

-- Research suggestions
CREATE TABLE research_suggestions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  suggestion_type VARCHAR(20) NOT NULL,
  title VARCHAR(500) NOT NULL,
  description TEXT NOT NULL,
  difficulty VARCHAR(20) NOT NULL,
  related_propositions JSONB DEFAULT '[]',
  potential_impact DECIMAL(3,2),
  generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  status VARCHAR(20) DEFAULT 'active', -- 'active', 'accepted', 'completed', 'dismissed'
  completed_at TIMESTAMP
);

-- Search index for full-text search
CREATE INDEX idx_propositions_search ON propositions USING gin(
  to_tsvector('english', title || ' ' || statement || ' ' || reasoning)
);

-- Performance indexes
CREATE INDEX idx_propositions_branch ON propositions(branch_id);
CREATE INDEX idx_propositions_status ON propositions(status);
CREATE INDEX idx_propositions_value ON propositions(value DESC);
CREATE INDEX idx_proposition_votes_proposition ON proposition_votes(proposition_id);
CREATE INDEX idx_proposition_votes_cycle ON proposition_votes(cycle);
CREATE INDEX idx_consistency_checks_branch ON consistency_checks(branch_id);
CREATE INDEX idx_philosophical_frameworks_academy ON philosophical_frameworks(academy_id, version DESC);
```

This philosophy and proposition management system provides the sophisticated knowledge synthesis framework needed for Peer Academy to build and evolve a shared worldview through systematic peer review, logical validation, and weighted consensus-building processes.