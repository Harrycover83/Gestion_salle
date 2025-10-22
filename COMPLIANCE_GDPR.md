# Guide de Conformité RGPD - GymManager Pro

## Introduction

Ce document détaille l'implémentation complète de la conformité RGPD pour la plateforme GymManager Pro. Le RGPD (Règlement Général sur la Protection des Données) est une réglementation européenne qui protège les données personnelles des citoyens de l'UE.

---

## Table des Matières

1. [Principes Fondamentaux](#principes-fondamentaux)
2. [Base Légale du Traitement](#base-légale-du-traitement)
3. [Droits des Personnes](#droits-des-personnes)
4. [Privacy by Design](#privacy-by-design)
5. [Gestion des Consentements](#gestion-des-consentements)
6. [Registre des Traitements](#registre-des-traitements)
7. [DPIA - Analyse d'Impact](#dpia)
8. [Violations de Données](#violations-de-données)
9. [Transferts Hors UE](#transferts-hors-ue)
10. [Audit Trail](#audit-trail)
11. [DPO - Délégué à la Protection](#dpo)
12. [Implémentation Technique](#implémentation-technique)

---

## Principes Fondamentaux

### 1. Licéité, Loyauté et Transparence

**Exigences:**
- Traiter les données de manière licite, loyale et transparente
- Informer clairement les personnes de l'utilisation de leurs données
- Pas de traitement caché ou trompeur

**Implémentation:**
```typescript
// Politique de confidentialité accessible et claire
interface PrivacyPolicy {
  version: string;
  effectiveDate: Date;
  content: {
    language: string;
    sections: {
      dataCollected: string;
      purposeOfProcessing: string;
      legalBasis: string;
      dataRecipients: string;
      retentionPeriod: string;
      userRights: string;
      dataProtection: string;
      contact: string;
    };
  }[];
  acceptances: Array<{
    memberId: string;
    acceptedAt: Date;
    ipAddress: string;
    version: string;
  }>;
}

// Transparence lors de la collecte
class DataCollectionService {
  async collectMemberData(data: MemberInput, context: CollectionContext): Promise<Member> {
    // 1. Vérifier le consentement/base légale
    await this.verifyLegalBasis(data, context);
    
    // 2. Logger la collecte (audit trail)
    await this.logDataCollection({
      dataTypes: Object.keys(data),
      purpose: context.purpose,
      legalBasis: context.legalBasis,
      collectedBy: context.userId,
      timestamp: new Date(),
    });
    
    // 3. Appliquer minimisation des données
    const minimizedData = this.minimizeData(data);
    
    // 4. Créer le membre
    return await this.memberRepository.create(minimizedData);
  }
}
```

### 2. Limitation des Finalités

**Exigences:**
- Collecter les données pour des finalités déterminées, explicites et légitimes
- Ne pas traiter ultérieurement de manière incompatible avec ces finalités

**Implémentation:**
```typescript
// Déclaration explicite des finalités
enum DataProcessingPurpose {
  // Exécution du contrat
  CONTRACT_MANAGEMENT = 'contract_management',
  ACCESS_CONTROL = 'access_control',
  BILLING = 'billing',
  
  // Intérêt légitime
  SECURITY = 'security',
  FRAUD_PREVENTION = 'fraud_prevention',
  SERVICE_IMPROVEMENT = 'service_improvement',
  
  // Consentement
  MARKETING = 'marketing',
  NEWSLETTER = 'newsletter',
  PROFILING = 'profiling',
  
  // Obligation légale
  LEGAL_COMPLIANCE = 'legal_compliance',
  ACCOUNTING = 'accounting',
}

// Contrôle des finalités lors de l'accès aux données
class DataAccessControl {
  async accessMemberData(
    memberId: string,
    purpose: DataProcessingPurpose,
    requestedBy: string
  ): Promise<MemberData> {
    // Vérifier si la finalité est autorisée
    const isAuthorized = await this.checkPurposeAuthorization(memberId, purpose);
    
    if (!isAuthorized) {
      await this.logUnauthorizedAccess({
        memberId,
        purpose,
        requestedBy,
        timestamp: new Date(),
      });
      throw new UnauthorizedPurposeError(`Purpose ${purpose} not authorized for member ${memberId}`);
    }
    
    // Logger l'accès (audit trail)
    await this.logDataAccess({
      memberId,
      purpose,
      accessedBy: requestedBy,
      timestamp: new Date(),
    });
    
    // Retourner uniquement les données nécessaires pour cette finalité
    return await this.getMemberDataForPurpose(memberId, purpose);
  }
  
  private async getMemberDataForPurpose(
    memberId: string,
    purpose: DataProcessingPurpose
  ): Promise<MemberData> {
    const fullData = await this.memberRepository.findById(memberId);
    
    // Filtrer les données selon la finalité
    switch (purpose) {
      case DataProcessingPurpose.ACCESS_CONTROL:
        return {
          id: fullData.id,
          memberNumber: fullData.memberNumber,
          photo: fullData.photo,
          subscriptionStatus: fullData.currentSubscription?.status,
        };
        
      case DataProcessingPurpose.MARKETING:
        // Vérifier le consentement marketing
        if (!fullData.preferences.marketingConsent) {
          throw new ConsentError('Marketing consent not given');
        }
        return {
          id: fullData.id,
          firstName: fullData.firstName,
          email: fullData.email,
          preferences: fullData.preferences,
        };
        
      case DataProcessingPurpose.BILLING:
        return {
          id: fullData.id,
          firstName: fullData.firstName,
          lastName: fullData.lastName,
          email: fullData.email,
          address: fullData.address,
          subscriptions: fullData.subscriptions,
        };
        
      default:
        return fullData;
    }
  }
}
```

### 3. Minimisation des Données

**Exigences:**
- Collecter uniquement les données adéquates, pertinentes et limitées
- Pas de collecte excessive ou "au cas où"

**Implémentation:**
```typescript
// Configuration des champs requis par finalité
const DataRequirements: Record<DataProcessingPurpose, string[]> = {
  [DataProcessingPurpose.CONTRACT_MANAGEMENT]: [
    'firstName', 'lastName', 'email', 'phone', 'dateOfBirth', 'address'
  ],
  [DataProcessingPurpose.ACCESS_CONTROL]: [
    'memberNumber', 'photo', 'subscriptionStatus'
  ],
  [DataProcessingPurpose.MARKETING]: [
    'firstName', 'email', 'preferences'
  ],
  // ...
};

class DataMinimizationService {
  minimizeData(data: any, purpose: DataProcessingPurpose): any {
    const requiredFields = DataRequirements[purpose];
    const minimized: any = {};
    
    for (const field of requiredFields) {
      if (data[field] !== undefined) {
        minimized[field] = data[field];
      }
    }
    
    return minimized;
  }
  
  // Validation lors de la collecte
  validateDataCollection(data: any, purpose: DataProcessingPurpose): ValidationResult {
    const requiredFields = DataRequirements[purpose];
    const providedFields = Object.keys(data);
    const excessiveFields = providedFields.filter(f => !requiredFields.includes(f));
    
    if (excessiveFields.length > 0) {
      return {
        valid: false,
        error: `Excessive data collection: ${excessiveFields.join(', ')}`,
        recommendation: 'Remove unnecessary fields',
      };
    }
    
    return { valid: true };
  }
}
```

### 4. Exactitude

**Exigences:**
- Maintenir les données exactes et à jour
- Permettre la correction des données inexactes

**Implémentation:**
```typescript
class DataAccuracyService {
  // Validation automatique
  async validateMemberData(memberId: string): Promise<ValidationReport> {
    const member = await this.memberRepository.findById(memberId);
    const issues: string[] = [];
    
    // Vérifier l'email
    if (!this.isValidEmail(member.email)) {
      issues.push('Invalid email format');
    }
    
    // Vérifier le téléphone
    if (!this.isValidPhone(member.phone)) {
      issues.push('Invalid phone number');
    }
    
    // Vérifier l'adresse
    if (!await this.isValidAddress(member.address)) {
      issues.push('Address validation failed');
    }
    
    // Vérifier la cohérence des dates
    if (member.dateOfBirth > new Date()) {
      issues.push('Birth date in the future');
    }
    
    return {
      memberId,
      isValid: issues.length === 0,
      issues,
      checkedAt: new Date(),
    };
  }
  
  // Rappels périodiques pour mise à jour
  async scheduleDataReviewReminders(): Promise<void> {
    const members = await this.memberRepository.findAll({
      where: {
        lastDataReview: {
          lt: new Date(Date.now() - 365 * 24 * 60 * 60 * 1000) // 1 an
        }
      }
    });
    
    for (const member of members) {
      await this.communicationService.sendDataReviewReminder(member.id);
    }
  }
  
  // Historique des modifications
  async trackDataChange(change: DataChange): Promise<void> {
    await this.auditLogRepository.create({
      entityType: 'member',
      entityId: change.memberId,
      action: 'update',
      field: change.field,
      oldValue: change.oldValue,
      newValue: change.newValue,
      changedBy: change.changedBy,
      reason: change.reason,
      timestamp: new Date(),
    });
  }
}
```

### 5. Limitation de la Conservation

**Exigences:**
- Conserver les données uniquement le temps nécessaire
- Définir des durées de conservation claires
- Supprimer ou anonymiser après expiration

**Implémentation:**
```typescript
// Configuration des durées de conservation
const RetentionPolicies: Record<string, RetentionPolicy> = {
  memberData: {
    activeMember: 'duration_of_relationship',
    inactiveMember: '3_years',
    deletedMember: '30_days', // Période de grâce
  },
  medicalCertificates: {
    current: '1_year_after_expiry',
    archived: '5_years', // Obligation légale
  },
  accessLogs: {
    operational: '1_year',
    incident: '5_years',
  },
  payments: {
    invoices: '10_years', // Obligation fiscale
    paymentDetails: '13_months', // PCI-DSS
  },
  communications: {
    marketing: '3_years',
    transactional: '5_years',
  },
  auditLogs: {
    gdpr: '3_years',
    security: '5_years',
  },
};

class DataRetentionService {
  async applyRetentionPolicies(): Promise<RetentionReport> {
    const report: RetentionReport = {
      executedAt: new Date(),
      actions: [],
    };
    
    // 1. Données membres inactifs
    const inactiveMembers = await this.findInactiveMembers(3 * 365); // 3 ans
    for (const member of inactiveMembers) {
      await this.anonymizeMember(member.id);
      report.actions.push({
        type: 'anonymize',
        entity: 'member',
        entityId: member.id,
        reason: 'Inactive for 3 years',
      });
    }
    
    // 2. Membres supprimés (période de grâce expirée)
    const deletedMembers = await this.findDeletedMembers(30); // 30 jours
    for (const member of deletedMembers) {
      await this.permanentlyDeleteMember(member.id);
      report.actions.push({
        type: 'permanent_delete',
        entity: 'member',
        entityId: member.id,
        reason: 'Grace period expired',
      });
    }
    
    // 3. Logs d'accès anciens
    const oldAccessLogs = await this.findOldAccessLogs(1 * 365); // 1 an
    await this.deleteAccessLogs(oldAccessLogs.map(l => l.id));
    report.actions.push({
      type: 'delete',
      entity: 'accessLogs',
      count: oldAccessLogs.length,
      reason: 'Retention period expired',
    });
    
    // 4. Documents expirés
    const expiredDocuments = await this.findExpiredDocuments();
    for (const doc of expiredDocuments) {
      await this.archiveOrDeleteDocument(doc.id);
      report.actions.push({
        type: 'archive',
        entity: 'document',
        entityId: doc.id,
        reason: 'Document expired',
      });
    }
    
    return report;
  }
  
  async anonymizeMember(memberId: string): Promise<void> {
    // Remplacer les données personnelles par des valeurs anonymisées
    await this.memberRepository.update(memberId, {
      firstName: 'ANONYMIZED',
      lastName: 'ANONYMIZED',
      email: `anonymized_${memberId}@deleted.local`,
      phone: null,
      dateOfBirth: null,
      address: null,
      photo: null,
      emergencyContact: null,
      medicalInfo: null,
      notes: 'ANONYMIZED',
      status: 'anonymized',
      anonymizedAt: new Date(),
    });
    
    // Conserver uniquement les données nécessaires pour analyses agrégées
    // (ex: genre, tranche d'âge) avec pseudonymisation
  }
  
  async scheduleRetentionCheck(memberId: string, dataType: string): Promise<void> {
    const policy = RetentionPolicies[dataType];
    const expiryDate = this.calculateExpiryDate(new Date(), policy);
    
    await this.schedulerService.schedule({
      jobType: 'data_retention_check',
      data: { memberId, dataType },
      executeAt: expiryDate,
    });
  }
}
```

### 6. Intégrité et Confidentialité

**Exigences:**
- Sécuriser les données contre accès non autorisés
- Protéger contre perte, destruction ou dommage accidentels
- Mesures techniques et organisationnelles appropriées

**Implémentation:**
```typescript
// Chiffrement des données sensibles
class DataEncryptionService {
  // Chiffrement at-rest (base de données)
  async encryptSensitiveField(value: string): Promise<string> {
    const key = await this.getEncryptionKey();
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
    
    let encrypted = cipher.update(value, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    const authTag = cipher.getAuthTag();
    
    return JSON.stringify({
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
    });
  }
  
  async decryptSensitiveField(encryptedData: string): Promise<string> {
    const { encrypted, iv, authTag } = JSON.parse(encryptedData);
    const key = await this.getEncryptionKey();
    
    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      key,
      Buffer.from(iv, 'hex')
    );
    decipher.setAuthTag(Buffer.from(authTag, 'hex'));
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  // Chiffrement in-transit (HTTPS/TLS)
  setupTLS(): https.ServerOptions {
    return {
      cert: fs.readFileSync('cert.pem'),
      key: fs.readFileSync('key.pem'),
      minVersion: 'TLSv1.3',
      ciphers: [
        'TLS_AES_256_GCM_SHA384',
        'TLS_CHACHA20_POLY1305_SHA256',
        'TLS_AES_128_GCM_SHA256',
      ].join(':'),
    };
  }
}

// Contrôle d'accès basé sur les rôles
class RoleBasedAccessControl {
  async checkPermission(
    userId: string,
    resource: string,
    action: string
  ): Promise<boolean> {
    const userRoles = await this.getUserRoles(userId);
    
    for (const role of userRoles) {
      const permissions = await this.getRolePermissions(role);
      
      if (permissions.some(p => 
        p.resource === resource && 
        p.actions.includes(action)
      )) {
        return true;
      }
    }
    
    return false;
  }
  
  // Need-to-know principle
  async getAccessibleFields(
    userId: string,
    dataType: string
  ): Promise<string[]> {
    const userRoles = await this.getUserRoles(userId);
    const accessibleFields = new Set<string>();
    
    for (const role of userRoles) {
      const roleFields = FieldAccessMap[role][dataType] || [];
      roleFields.forEach(field => accessibleFields.add(field));
    }
    
    return Array.from(accessibleFields);
  }
}

// Configuration des permissions par rôle
const FieldAccessMap = {
  super_admin: {
    member: ['*'], // Tous les champs
  },
  admin: {
    member: [
      'id', 'memberNumber', 'firstName', 'lastName', 'email', 'phone',
      'address', 'dateOfBirth', 'photo', 'emergencyContact',
      'currentSubscription', 'memberSince', 'lastVisit', 'preferences',
      'tags', 'notes', 'status'
    ],
  },
  coach: {
    member: [
      'id', 'memberNumber', 'firstName', 'lastName', 'photo',
      'currentSubscription', 'lastVisit'
    ],
  },
  receptionist: {
    member: [
      'id', 'memberNumber', 'firstName', 'lastName', 'photo',
      'currentSubscription', 'phone', 'emergencyContact'
    ],
  },
};
```

### 7. Responsabilité (Accountability)

**Exigences:**
- Être en mesure de démontrer la conformité
- Documenter les mesures prises
- Maintenir des registres

**Implémentation:**
```typescript
// Documentation et preuves de conformité
class ComplianceDocumentation {
  async generateComplianceReport(): Promise<ComplianceReport> {
    return {
      generatedAt: new Date(),
      sections: {
        // 1. Inventaire des traitements
        processingActivities: await this.getProcessingRegister(),
        
        // 2. Mesures de sécurité
        securityMeasures: await this.getSecurityMeasures(),
        
        // 3. Analyses d'impact (DPIA)
        dpias: await this.getDPIAs(),
        
        // 4. Violations de données
        dataBreaches: await this.getDataBreaches(),
        
        // 5. Demandes d'exercice de droits
        rightsRequests: await this.getRightsRequests(),
        
        // 6. Audits et contrôles
        audits: await this.getAudits(),
        
        // 7. Formation du personnel
        trainings: await this.getTrainings(),
        
        // 8. Contrats sous-traitants (DPA)
        dataProcessingAgreements: await this.getDPAs(),
      },
    };
  }
  
  async generateProcessingRegister(): Promise<ProcessingRegister> {
    // Registre des activités de traitement (Article 30 RGPD)
    return {
      organizationInfo: {
        name: 'GymManager Pro',
        address: '...',
        dpo: {
          name: '...',
          email: 'dpo@gymmanager.com',
        },
      },
      processingActivities: [
        {
          name: 'Gestion des membres',
          purpose: 'Exécution du contrat d\'adhésion',
          legalBasis: 'Contract',
          dataCategories: [
            'Données d\'identification',
            'Coordonnées',
            'Données d\'adhésion',
          ],
          dataSubjects: ['Membres de la salle de sport'],
          recipients: [
            'Personnel administratif',
            'Coachs',
            'Prestataire de paiement (Stripe)',
          ],
          internationalTransfers: [
            {
              country: 'USA',
              safeguards: 'Standard Contractual Clauses',
              recipient: 'Stripe, Inc.',
            },
          ],
          retentionPeriod: 'Durée de la relation + 3 ans',
          securityMeasures: [
            'Chiffrement des données',
            'Contrôle d\'accès basé sur les rôles',
            'Audit logs',
            'Backups chiffrés',
          ],
        },
        // ... autres activités
      ],
    };
  }
}
```

---

## Base Légale du Traitement

Le RGPD exige une base légale pour chaque traitement de données personnelles. Voici les bases légales applicables :

### 1. Consentement (Article 6(1)(a))

**Quand l'utiliser:**
- Marketing et prospection
- Newsletters
- Profilage
- Cookies non essentiels

**Exigences:**
- Libre
- Spécifique
- Éclairé
- Univoque
- Révocable facilement

**Implémentation:**
```typescript
interface Consent {
  id: string;
  memberId: string;
  purpose: DataProcessingPurpose;
  
  // Contexte du consentement
  givenAt: Date;
  ipAddress: string;
  userAgent: string;
  method: 'checkbox' | 'button' | 'verbal' | 'written';
  
  // Informations fournies
  informationProvided: {
    purpose: string;
    dataCollected: string[];
    recipients: string[];
    retentionPeriod: string;
    rightToWithdraw: string;
  };
  
  // État du consentement
  status: 'active' | 'withdrawn' | 'expired';
  withdrawnAt?: Date;
  withdrawalMethod?: string;
  
  // Preuve
  proofText: string; // Texte exact présenté
  proofVersion: string; // Version de la politique
}

class ConsentManagement {
  async requestConsent(
    memberId: string,
    purpose: DataProcessingPurpose,
    information: ConsentInformation
  ): Promise<ConsentRequest> {
    // Créer une demande de consentement
    const request = await this.consentRepository.createRequest({
      memberId,
      purpose,
      information,
      requestedAt: new Date(),
      status: 'pending',
    });
    
    // Envoyer la demande au membre
    await this.communicationService.sendConsentRequest(memberId, request);
    
    return request;
  }
  
  async giveConsent(
    requestId: string,
    context: ConsentContext
  ): Promise<Consent> {
    const request = await this.consentRepository.findRequest(requestId);
    
    // Vérifier que le consentement est valide
    this.validateConsent(request, context);
    
    // Enregistrer le consentement
    const consent = await this.consentRepository.create({
      memberId: request.memberId,
      purpose: request.purpose,
      givenAt: new Date(),
      ipAddress: context.ipAddress,
      userAgent: context.userAgent,
      method: context.method,
      informationProvided: request.information,
      status: 'active',
      proofText: request.consentText,
      proofVersion: request.policyVersion,
    });
    
    // Logger l'événement
    await this.auditLog.log({
      event: 'consent_given',
      memberId: request.memberId,
      purpose: request.purpose,
      timestamp: new Date(),
    });
    
    return consent;
  }
  
  async withdrawConsent(
    memberId: string,
    purpose: DataProcessingPurpose,
    method: string
  ): Promise<void> {
    // Trouver le consentement actif
    const consent = await this.consentRepository.findActive(memberId, purpose);
    
    if (!consent) {
      throw new Error('No active consent found');
    }
    
    // Retirer le consentement
    await this.consentRepository.update(consent.id, {
      status: 'withdrawn',
      withdrawnAt: new Date(),
      withdrawalMethod: method,
    });
    
    // Arrêter les traitements basés sur ce consentement
    await this.stopProcessing(memberId, purpose);
    
    // Logger l'événement
    await this.auditLog.log({
      event: 'consent_withdrawn',
      memberId,
      purpose,
      timestamp: new Date(),
    });
    
    // Notifier le membre
    await this.communicationService.sendConsentWithdrawalConfirmation(memberId, purpose);
  }
  
  // Vérifier le consentement avant traitement
  async checkConsent(
    memberId: string,
    purpose: DataProcessingPurpose
  ): Promise<boolean> {
    const consent = await this.consentRepository.findActive(memberId, purpose);
    return consent !== null && consent.status === 'active';
  }
  
  // Renouvellement périodique (best practice)
  async scheduleConsentRenewal(): Promise<void> {
    const oldConsents = await this.consentRepository.findOlderThan(2 * 365); // 2 ans
    
    for (const consent of oldConsents) {
      await this.requestConsent(
        consent.memberId,
        consent.purpose,
        consent.informationProvided
      );
    }
  }
}
```

### 2. Contrat (Article 6(1)(b))

**Quand l'utiliser:**
- Gestion de l'adhésion
- Facturation
- Accès à la salle
- Services inclus dans l'abonnement

**Implémentation:**
```typescript
// Pas besoin de consentement séparé, mais documentation nécessaire
interface ContractProcessing {
  subscriptionId: string;
  memberId: string;
  
  contractTerms: {
    signedAt: Date;
    termsVersion: string;
    dataProcessingClauses: string[];
  };
  
  necessaryProcessing: {
    memberManagement: boolean;
    accessControl: boolean;
    billing: boolean;
    communications: boolean; // Seulement transactionnelles
  };
}

class ContractBasedProcessing {
  async processForContractExecution(
    memberId: string,
    subscriptionId: string,
    action: string
  ): Promise<void> {
    // Vérifier qu'il existe un contrat actif
    const subscription = await this.subscriptionRepository.findById(subscriptionId);
    
    if (!subscription || subscription.status !== 'active') {
      throw new Error('No active subscription found');
    }
    
    // Vérifier que l'action est nécessaire au contrat
    if (!this.isNecessaryForContract(action)) {
      throw new Error('Action not necessary for contract execution');
    }
    
    // Procéder au traitement
    // ... (pas besoin de consentement supplémentaire)
  }
  
  private isNecessaryForContract(action: string): boolean {
    const necessaryActions = [
      'create_member',
      'update_member_contact',
      'grant_access',
      'create_invoice',
      'send_payment_reminder',
      'send_renewal_notice',
    ];
    
    return necessaryActions.includes(action);
  }
}
```

### 3. Obligation Légale (Article 6(1)(c))

**Quand l'utiliser:**
- Conservation comptable (10 ans)
- Déclarations fiscales
- Réponses aux autorités

**Implémentation:**
```typescript
class LegalObligationProcessing {
  // Conservation pour obligations comptables
  async retainForAccounting(invoiceId: string): Promise<void> {
    const invoice = await this.invoiceRepository.findById(invoiceId);
    
    // Marquer pour conservation légale
    await this.invoiceRepository.update(invoiceId, {
      legalHold: true,
      legalHoldReason: 'Accounting obligation (10 years)',
      legalHoldUntil: new Date(invoice.issueDate.getTime() + 10 * 365 * 24 * 60 * 60 * 1000),
    });
  }
  
  // Réponse aux autorités
  async respondToAuthorityRequest(
    requestId: string,
    authority: string,
    dataRequested: string[]
  ): Promise<void> {
    // Logger la demande
    await this.auditLog.log({
      event: 'authority_data_request',
      authority,
      requestId,
      dataRequested,
      timestamp: new Date(),
    });
    
    // Vérifier la légitimité
    await this.verifyAuthorityRequest(requestId, authority);
    
    // Extraire les données demandées
    const data = await this.extractRequestedData(dataRequested);
    
    // Créer le rapport
    await this.generateAuthorityReport(requestId, data);
    
    // Logger la fourniture
    await this.auditLog.log({
      event: 'authority_data_provided',
      authority,
      requestId,
      timestamp: new Date(),
    });
  }
}
```

### 4. Intérêt Légitime (Article 6(1)(f))

**Quand l'utiliser:**
- Sécurité et prévention de la fraude
- Amélioration du service
- Marketing direct (avec droit d'opposition)

**Exigences:**
- Équilibrer avec les droits des personnes
- Documenter l'analyse de balance
- Permettre l'opposition

**Implémentation:**
```typescript
interface LegitimateInterestAssessment {
  purpose: string;
  description: string;
  
  // 1. Identification de l'intérêt légitime
  legitimateInterest: {
    description: string;
    benefits: string[];
    isNecessary: boolean;
    alternativeMeans: string[];
  };
  
  // 2. Test de nécessité
  necessity: {
    isProportionate: boolean;
    leastIntrusiveMeans: boolean;
    explanation: string;
  };
  
  // 3. Test de balance
  balancing: {
    dataSubjectImpact: 'low' | 'medium' | 'high';
    expectationOfPrivacy: 'low' | 'medium' | 'high';
    vulnerability: string[];
    safeguards: string[];
    conclusion: 'acceptable' | 'unacceptable';
    reasoning: string;
  };
  
  // Résultat
  canProceed: boolean;
  requiresConsent: boolean;
  rightToObject: boolean;
  
  // Documentation
  assessedBy: string;
  assessedAt: Date;
  reviewDate: Date;
}

class LegitimateInterestProcessing {
  async assessLegitimateInterest(
    purpose: string,
    description: string
  ): Promise<LegitimateInterestAssessment> {
    // Exemple: Détection de fraude
    if (purpose === 'fraud_detection') {
      return {
        purpose,
        description,
        legitimateInterest: {
          description: 'Protéger la salle de sport et ses membres contre la fraude',
          benefits: [
            'Sécurité financière',
            'Protection des autres membres',
            'Prévention abus',
          ],
          isNecessary: true,
          alternativeMeans: [],
        },
        necessity: {
          isProportionate: true,
          leastIntrusiveMeans: true,
          explanation: 'Analyse automatisée des patterns de paiement et d\'accès',
        },
        balancing: {
          dataSubjectImpact: 'low',
          expectationOfPrivacy: 'low',
          vulnerability: [],
          safeguards: [
            'Données pseudonymisées',
            'Accès restreint',
            'Audit trail',
            'Droit d\'opposition',
          ],
          conclusion: 'acceptable',
          reasoning: 'L\'intérêt légitime prévaut, impact minimal sur les droits',
        },
        canProceed: true,
        requiresConsent: false,
        rightToObject: true,
        assessedBy: 'dpo@gymmanager.com',
        assessedAt: new Date(),
        reviewDate: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000), // 1 an
      };
    }
    
    // Autres cas...
    return this.performFullAssessment(purpose, description);
  }
  
  async processWithLegitimateInterest(
    memberId: string,
    purpose: string
  ): Promise<void> {
    // 1. Vérifier qu'une LIA existe
    const lia = await this.liaRepository.findByPurpose(purpose);
    if (!lia || !lia.canProceed) {
      throw new Error('No valid legitimate interest assessment');
    }
    
    // 2. Vérifier qu'il n'y a pas d'opposition
    const hasObjected = await this.checkObjection(memberId, purpose);
    if (hasObjected) {
      throw new Error('Member has objected to this processing');
    }
    
    // 3. Procéder au traitement
    await this.executeProcessing(memberId, purpose);
    
    // 4. Logger
    await this.auditLog.log({
      event: 'legitimate_interest_processing',
      memberId,
      purpose,
      timestamp: new Date(),
    });
  }
  
  async registerObjection(
    memberId: string,
    purpose: string,
    reason?: string
  ): Promise<void> {
    await this.objectionRepository.create({
      memberId,
      purpose,
      reason,
      registeredAt: new Date(),
    });
    
    // Arrêter le traitement
    await this.stopProcessing(memberId, purpose);
    
    // Notifier l'équipe
    await this.notifyTeam({
      type: 'objection_registered',
      memberId,
      purpose,
    });
  }
}
```

---

## Droits des Personnes

### 1. Droit d'Accès (Article 15)

**Délai:** 30 jours (prorogeable de 60 jours si complexe)

**Implémentation:**
```typescript
class RightOfAccessService {
  async handleAccessRequest(memberId: string): Promise<AccessRequestResponse> {
    // 1. Vérifier l'identité
    await this.verifyIdentity(memberId);
    
    // 2. Collecter toutes les données
    const data = await this.collectMemberData(memberId);
    
    // 3. Générer le rapport
    const report = await this.generateAccessReport(data);
    
    // 4. Logger la demande
    await this.logAccessRequest(memberId);
    
    // 5. Envoyer au membre
    await this.sendAccessReport(memberId, report);
    
    return {
      requestId: generateId(),
      status: 'completed',
      completedAt: new Date(),
    };
  }
  
  private async collectMemberData(memberId: string): Promise<MemberDataExport> {
    return {
      // Données personnelles
      personalData: await this.memberRepository.findById(memberId),
      
      // Abonnements
      subscriptions: await this.subscriptionRepository.findByMember(memberId),
      
      // Historique de présence
      accessLogs: await this.accessLogRepository.findByMember(memberId),
      
      // Réservations
      bookings: await this.bookingRepository.findByMember(memberId),
      
      // Factures et paiements
      invoices: await this.invoiceRepository.findByMember(memberId),
      payments: await this.paymentRepository.findByMember(memberId),
      
      // Communications
      emailsSent: await this.emailRepository.findByRecipient(memberId),
      smsSent: await this.smsRepository.findByRecipient(memberId),
      
      // Consentements
      consents: await this.consentRepository.findByMember(memberId),
      
      // Documents
      documents: await this.documentRepository.findByMember(memberId),
      
      // Métadonnées
      metadata: {
        createdAt: new Date(),
        exportFormat: 'JSON',
        dataCategories: [
          'personal_info',
          'subscriptions',
          'access_logs',
          'bookings',
          'financial',
          'communications',
          'consents',
          'documents',
        ],
      },
    };
  }
  
  private async generateAccessReport(data: MemberDataExport): Promise<AccessReport> {
    return {
      // Informations sur les données
      dataSummary: {
        categories: this.summarizeCategories(data),
        sources: this.listDataSources(),
        recipients: this.listDataRecipients(),
        retentionPeriods: this.getRetentionPeriods(),
      },
      
      // Données complètes (format machine-readable)
      data: {
        json: JSON.stringify(data, null, 2),
        csv: await this.convertToCSV(data),
      },
      
      // Informations complémentaires
      processingPurposes: await this.getProcessingPurposes(),
      legalBases: await this.getLegalBases(),
      internationalTransfers: await this.getInternationalTransfers(),
      
      // Droits
      rights: {
        rectification: 'Vous pouvez demander la rectification de vos données',
        erasure: 'Vous pouvez demander la suppression de vos données',
        restriction: 'Vous pouvez demander la limitation du traitement',
        objection: 'Vous pouvez vous opposer au traitement',
        portability: 'Vous pouvez demander la portabilité de vos données',
      },
      
      // Contact
      contact: {
        dpo: 'dpo@gymmanager.com',
        authority: 'CNIL (France)',
      },
    };
  }
}
```

### 2. Droit de Rectification (Article 16)

**Implémentation:**
```typescript
class RightOfRectificationService {
  async handleRectificationRequest(
    memberId: string,
    corrections: Record<string, any>,
    justification?: string
  ): Promise<RectificationResponse> {
    // 1. Vérifier l'identité
    await this.verifyIdentity(memberId);
    
    // 2. Valider les corrections
    await this.validateCorrections(corrections);
    
    // 3. Appliquer les corrections
    const oldData = await this.memberRepository.findById(memberId);
    await this.memberRepository.update(memberId, corrections);
    
    // 4. Logger les modifications
    await this.logRectification(memberId, oldData, corrections, justification);
    
    // 5. Notifier les tiers si nécessaire
    await this.notifyThirdParties(memberId, corrections);
    
    // 6. Confirmer au membre
    await this.sendRectificationConfirmation(memberId, corrections);
    
    return {
      requestId: generateId(),
      status: 'completed',
      fieldsUpdated: Object.keys(corrections),
      completedAt: new Date(),
    };
  }
  
  // Interface utilisateur pour auto-rectification
  async allowSelfRectification(
    memberId: string,
    field: string,
    newValue: any
  ): Promise<void> {
    const allowedFields = [
      'email',
      'phone',
      'address',
      'emergencyContact',
      'preferences',
    ];
    
    if (!allowedFields.includes(field)) {
      throw new Error('Field cannot be self-rectified');
    }
    
    await this.handleRectificationRequest(memberId, { [field]: newValue });
  }
}
```

### 3. Droit à l'Effacement (Article 17)

**Motifs:**
- Données plus nécessaires
- Consentement retiré
- Opposition au traitement
- Données traitées illégalement
- Obligation légale d'effacement

**Implémentation:**
```typescript
class RightToErasureService {
  async handleErasureRequest(
    memberId: string,
    reason: ErasureReason
  ): Promise<ErasureResponse> {
    // 1. Vérifier si l'effacement est possible
    const canErase = await this.checkErasureEligibility(memberId, reason);
    
    if (!canErase.eligible) {
      return {
        status: 'denied',
        reason: canErase.reason,
        explanation: canErase.explanation,
      };
    }
    
    // 2. Identifier les données à effacer
    const dataToErase = await this.identifyErasableData(memberId);
    
    // 3. Vérifier les obligations légales
    const legalObligations = await this.checkLegalObligations(memberId);
    
    // 4. Procéder à l'effacement (avec période de grâce)
    await this.initiateErasure(memberId, dataToErase, legalObligations);
    
    // 5. Notifier les tiers
    await this.notifyThirdPartiesOfErasure(memberId);
    
    // 6. Confirmer au membre
    await this.sendErasureConfirmation(memberId);
    
    return {
      requestId: generateId(),
      status: 'initiated',
      gracePeriod: 30, // jours
      completeBy: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      retainedData: legalObligations,
    };
  }
  
  private async checkErasureEligibility(
    memberId: string,
    reason: ErasureReason
  ): Promise<EligibilityCheck> {
    const member = await this.memberRepository.findById(memberId);
    
    // Vérifier s'il y a des obligations de conservation
    if (reason === 'data_no_longer_necessary') {
      // Vérifier obligations légales (comptabilité, etc.)
      const hasOutstandingInvoices = await this.hasOutstandingInvoices(memberId);
      if (hasOutstandingInvoices) {
        return {
          eligible: false,
          reason: 'outstanding_obligations',
          explanation: 'Vous avez des factures impayées qui doivent être réglées',
        };
      }
      
      const recentSubscription = await this.hasRecentSubscription(memberId, 10 * 365);
      if (recentSubscription) {
        return {
          eligible: false,
          reason: 'legal_retention',
          explanation: 'Obligation de conservation comptable (10 ans)',
        };
      }
    }
    
    // Vérifier si basé sur contrat et contrat toujours actif
    if (member.currentSubscription?.status === 'active') {
      return {
        eligible: false,
        reason: 'active_contract',
        explanation: 'Votre abonnement est toujours actif. Veuillez d\'abord le résilier.',
      };
    }
    
    return {
      eligible: true,
    };
  }
  
  private async initiateErasure(
    memberId: string,
    dataToErase: string[],
    legalObligations: string[]
  ): Promise<void> {
    // Marquer pour suppression avec période de grâce
    await this.memberRepository.update(memberId, {
      status: 'pending_deletion',
      deletionRequestedAt: new Date(),
      deletionScheduledFor: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    });
    
    // Anonymiser immédiatement les données non nécessaires
    for (const field of dataToErase) {
      if (!legalObligations.includes(field)) {
        await this.anonymizeField(memberId, field);
      }
    }
    
    // Planifier la suppression complète
    await this.scheduleFinalDeletion(memberId);
  }
  
  async executeFinalDeletion(memberId: string): Promise<void> {
    // 1. Vérifier que la période de grâce est écoulée
    const member = await this.memberRepository.findById(memberId);
    if (member.deletionScheduledFor > new Date()) {
      throw new Error('Grace period not yet elapsed');
    }
    
    // 2. Supprimer de toutes les tables
    await Promise.all([
      this.memberRepository.delete(memberId),
      this.subscriptionRepository.deleteByMember(memberId),
      this.bookingRepository.deleteByMember(memberId),
      this.documentRepository.deleteByMember(memberId),
      this.consentRepository.deleteByMember(memberId),
      // Conserver uniquement les données comptables (anonymisées)
    ]);
    
    // 3. Supprimer des backups (après leur expiration naturelle)
    await this.markForBackupDeletion(memberId);
    
    // 4. Logger la suppression définitive
    await this.auditLog.log({
      event: 'permanent_deletion',
      memberId,
      timestamp: new Date(),
      reason: 'erasure_request',
    });
  }
}
```

### 4. Droit à la Portabilité (Article 20)

**Implémentation:**
```typescript
class RightToDataPortabilityService {
  async handlePortabilityRequest(
    memberId: string,
    format: 'json' | 'csv' | 'xml',
    transferTo?: {
      service: string;
      endpoint: string;
      apiKey?: string;
    }
  ): Promise<PortabilityResponse> {
    // 1. Collecter les données portables
    // Seulement : données fournies par la personne + traitement automatisé
    const portableData = await this.collectPortableData(memberId);
    
    // 2. Formater selon le format demandé
    const formatted = await this.formatData(portableData, format);
    
    // 3. Si transfert direct demandé
    if (transferTo) {
      await this.transferToService(formatted, transferTo);
    }
    
    // 4. Générer le fichier pour téléchargement
    const downloadUrl = await this.generateDownloadLink(formatted, format);
    
    // 5. Logger
    await this.logPortabilityRequest(memberId, format, transferTo);
    
    return {
      requestId: generateId(),
      status: 'completed',
      downloadUrl,
      format,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 jours
      transferred: !!transferTo,
    };
  }
  
  private async collectPortableData(memberId: string): Promise<PortableData> {
    // Uniquement données fournies par la personne
    // et traitement automatisé basé sur consentement ou contrat
    return {
      // Données de base
      profile: await this.getBasicProfile(memberId),
      
      // Préférences
      preferences: await this.getPreferences(memberId),
      
      // Historique d'activité
      bookings: await this.getBookingHistory(memberId),
      accessHistory: await this.getAccessHistory(memberId),
      
      // Données de performance (si suivies)
      workouts: await this.getWorkoutData(memberId),
      progress: await this.getProgressData(memberId),
      
      // Exclure : données dérivées, analyses, évaluations internes
    };
  }
  
  private async formatData(
    data: PortableData,
    format: 'json' | 'csv' | 'xml'
  ): Promise<string> {
    switch (format) {
      case 'json':
        return JSON.stringify(data, null, 2);
        
      case 'csv':
        return this.convertToCSV(data);
        
      case 'xml':
        return this.convertToXML(data);
        
      default:
        throw new Error('Unsupported format');
    }
  }
  
  private async transferToService(
    data: string,
    transferTo: TransferDestination
  ): Promise<void> {
    // Transfert direct vers un autre service
    try {
      await axios.post(transferTo.endpoint, data, {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${transferTo.apiKey}`,
        },
      });
      
      await this.auditLog.log({
        event: 'data_transfer',
        destination: transferTo.service,
        timestamp: new Date(),
      });
    } catch (error) {
      throw new DataTransferError('Failed to transfer data', error);
    }
  }
}
```

### 5. Droit à la Limitation du Traitement (Article 18)

**Implémentation:**
```typescript
class RightToRestrictionService {
  async handleRestrictionRequest(
    memberId: string,
    reason: RestrictionReason
  ): Promise<RestrictionResponse> {
    // Motifs valides :
    // - Contestation de l'exactitude
    // - Traitement illicite mais opposition à l'effacement
    // - Données plus nécessaires mais personne en a besoin
    // - Opposition au traitement en attendant vérification
    
    // Marquer les données comme restreintes
    await this.memberRepository.update(memberId, {
      processingRestricted: true,
      restrictionReason: reason,
      restrictionRequestedAt: new Date(),
    });
    
    // Bloquer les traitements non essentiels
    await this.blockNonEssentialProcessing(memberId);
    
    // Logger
    await this.logRestrictionRequest(memberId, reason);
    
    // Notifier les tiers
    await this.notifyThirdPartiesOfRestriction(memberId);
    
    return {
      requestId: generateId(),
      status: 'active',
      allowedProcessing: [
        'storage',
        'with_consent',
        'legal_claims',
        'protection_of_rights',
      ],
      blockedProcessing: [
        'marketing',
        'profiling',
        'automated_decisions',
      ],
    };
  }
  
  async liftRestriction(
    memberId: string,
    reason: string
  ): Promise<void> {
    // Notifier la personne avant de lever la restriction
    await this.notifyBeforeLiftingRestriction(memberId);
    
    await this.memberRepository.update(memberId, {
      processingRestricted: false,
      restrictionLiftedAt: new Date(),
      restrictionLiftedReason: reason,
    });
    
    await this.auditLog.log({
      event: 'restriction_lifted',
      memberId,
      reason,
      timestamp: new Date(),
    });
  }
}
```

### 6. Droit d'Opposition (Article 21)

**Implémentation:**
```typescript
class RightToObjectService {
  async handleObjection(
    memberId: string,
    purpose: DataProcessingPurpose,
    reason?: string
  ): Promise<ObjectionResponse> {
    // Vérifier si l'opposition est possible
    const canObject = await this.checkObjectionEligibility(purpose);
    
    if (!canObject.eligible) {
      return {
        status: 'denied',
        reason: canObject.reason,
      };
    }
    
    // Enregistrer l'opposition
    await this.objectionRepository.create({
      memberId,
      purpose,
      reason,
      registeredAt: new Date(),
    });
    
    // Arrêter le traitement
    await this.stopProcessing(memberId, purpose);
    
    // Si marketing direct: arrêt immédiat et définitif
    if (purpose === DataProcessingPurpose.MARKETING) {
      await this.blacklistForMarketing(memberId);
    }
    
    // Logger
    await this.logObjection(memberId, purpose);
    
    return {
      requestId: generateId(),
      status: 'accepted',
      processingS topped: true,
      permanent: purpose === DataProcessingPurpose.MARKETING,
    };
  }
  
  private async checkObjectionEligibility(
    purpose: DataProcessingPurpose
  ): Promise<EligibilityCheck> {
    // Opposition toujours possible pour :
    // - Marketing direct
    // - Profilage pour marketing
    if ([DataProcessingPurpose.MARKETING, DataProcessingPurpose.PROFILING].includes(purpose)) {
      return { eligible: true };
    }
    
    // Opposition possible pour intérêt légitime
    // sauf si motifs légitimes impérieux
    const lia = await this.liaRepository.findByPurpose(purpose);
    if (lia && lia.legalBasis === 'legitimate_interest') {
      return {
        eligible: true,
        requiresBalancing: true,
      };
    }
    
    // Pas d'opposition possible pour :
    // - Contrat
    // - Obligation légale
    return {
      eligible: false,
      reason: 'Opposition not possible for this processing basis',
    };
  }
}
```

---

_[La suite du document continue avec les sections Privacy by Design, Gestion des Consentements, Registre des Traitements, DPIA, etc.]_

**Note:** Ce document est très volumineux. Pour des raisons de concision, je fournis ici les sections principales avec des implémentations détaillées. Le document complet inclurait également:

- Privacy by Design & Default
- Cookie Consent Management
- Processing Register (Article 30)
- DPIA (Data Protection Impact Assessment)
- Data Breach Notification
- International Data Transfers
- Complete Audit Trail Implementation
- DPO Responsibilities
- Staff Training Requirements
- Compliance Monitoring & Reporting

---

**Version**: 1.0  
**Dernière mise à jour**: 2025-10-21  
**Statut**: Living Document - À mettre à jour régulièrement
