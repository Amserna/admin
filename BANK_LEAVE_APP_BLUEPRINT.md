# Application bancaire de gestion des congés (Laravel + MySQL)

## 1) Architecture cible (production bancaire)

**Stack obligatoire**
- Backend: Laravel 11+
- Base de données: MySQL 8+
- Queue: Redis + Laravel Horizon
- Cache: Redis
- Notifications temps réel: Laravel Reverb (ou Pusher compatible)
- Auth: Laravel Sanctum (API) + session sécurisée (web)

**Architecture en couches**
- `Controllers` : HTTP, validation, mapping DTO
- `Services` : orchestration métier + transactions
- `Repositories` : accès données encapsulé
- `Models` : relations Eloquent + casts + scopes
- `Policies` : autorisations de transition d’état
- `Observers` : audit automatique
- `Jobs/Notifications` : asynchrone (emails, in-app, websocket)

## 2) Structure projet Laravel recommandée

```text
app/
  Enums/
    LeaveRequestStatus.php
    LeaveApprovalDecision.php
    RoleCode.php
  Http/
    Controllers/
      AuthController.php
      DashboardController.php
      LeaveRequestController.php
      LeaveApprovalController.php
      ExportController.php
  Models/
    User.php
    Role.php
    Service.php
    Department.php
    Position.php
    LeaveRequest.php
    LeaveApproval.php
    LeaveBalance.php
    ApprovalLevel.php
    AuditLog.php
  Policies/
    LeaveRequestPolicy.php
  Repositories/
    Contracts/
      LeaveRequestRepositoryInterface.php
      LeaveApprovalRepositoryInterface.php
    Eloquent/
      LeaveRequestRepository.php
      LeaveApprovalRepository.php
  Services/
    LeaveWorkflowService.php
    LeaveRequestService.php
    AuditService.php
    NotificationService.php
  Jobs/
    SendLeaveStatusNotificationJob.php
  Notifications/
    LeaveStatusChangedNotification.php
  Observers/
    LeaveRequestObserver.php
config/
  workflow.php
database/
  migrations/
  seeders/
routes/
  web.php
  api.php
tests/
  Feature/LeaveWorkflowTest.php
  Unit/LeaveStateTransitionTest.php
```

## 3) Workflow hiérarchique imposé

Ordre strict:

1. Employé crée la demande (`CREATED`)
2. Chef de Service (`PENDING_SERVICE_HEAD`)
3. Supérieurs hiérarchiques (`PENDING_HIERARCHY`)
4. DGA avis consultatif (`PENDING_DGA_OPINION`)
5. DG avis consultatif (`PENDING_DG_OPINION`)
6. RH décision finale (`PENDING_HR_DECISION`)
7. Système met à jour solde + notifie (`APPROVED` / `REJECTED`)

### Règles non négociables
- Rejet par **Chef de Service** ou **Supérieurs hiérarchiques** => arrêt immédiat (`REJECTED`).
- DGA et DG = avis consultatifs, non bloquants.
- Seule RH clôture définitivement.
- Historique complet et immuable des validations.
- Suppression interdite sur validations (archivage logique ailleurs si nécessaire).
- L’employé voit en continu son statut (**En attente / Validé / Rejeté**) à chaque étape.

## 4) États (enum PHP)

```php
<?php

namespace App\Enums;

enum LeaveRequestStatus: string
{
    case CREATED = 'CREATED';
    case PENDING_SERVICE_HEAD = 'PENDING_SERVICE_HEAD';
    case PENDING_HIERARCHY = 'PENDING_HIERARCHY';
    case PENDING_DGA_OPINION = 'PENDING_DGA_OPINION';
    case PENDING_DG_OPINION = 'PENDING_DG_OPINION';
    case PENDING_HR_DECISION = 'PENDING_HR_DECISION';
    case APPROVED = 'APPROVED';
    case REJECTED = 'REJECTED';
}
```

## 5) Migrations détaillées (résumé exécutable)

> Toutes les tables critiques incluent: `created_at`, `updated_at`, `deleted_at` (soft delete), index de recherche, FK.

### `roles`
- `id`, `code` (unique), `name`, timestamps, softDeletes

### `users`
- `id`, `role_id`, `service_id`, `department_id`, `position_id`
- `manager_id` (auto-référence pour la hiérarchie)
- `employee_number` (unique), `name`, `email` (unique), `password`
- `is_active`, `last_login_at`, timestamps, softDeletes
- Index: `(role_id)`, `(service_id)`, `(department_id)`, `(manager_id)`, `(is_active)`

### `services`
- `id`, `code` unique, `name`, timestamps, softDeletes

### `departments`
- `id`, `service_id`, `code` unique, `name`, timestamps, softDeletes

### `positions`
- `id`, `department_id`, `code` unique, `name`, `grade`, timestamps, softDeletes

### `leave_requests`
- `id`, `employee_id`, `leave_type`, `start_date`, `end_date`, `days_requested`
- `reason`, `status` (enum), `current_level_order`, `final_decision_by` nullable
- `final_decision_at` nullable, timestamps, softDeletes
- Index: `(employee_id, status)`, `(status)`, `(start_date, end_date)`, `(current_level_order)`

### `approval_levels`
- `id`, `code` unique (`SERVICE_HEAD`, `HIERARCHY`, `DGA`, `DG`, `HR`)
- `label`, `level_order` unique, `is_consultative` bool
- `is_active`, timestamps, softDeletes

### `leave_approvals`
- `id`, `leave_request_id`, `approval_level_id`, `approver_id`
- `decision` enum (`APPROVED`,`REJECTED`,`OPINION_POSITIVE`,`OPINION_NEGATIVE`)
- `comment`, `decided_at`
- timestamps
- **Sans soft delete** pour immutabilité (aucune suppression)
- Contrainte: index `(leave_request_id, approval_level_id, approver_id)`

### `leave_balances`
- `id`, `employee_id` unique, `year`, `total_days`, `used_days`, `remaining_days`
- `last_adjusted_by`, `last_adjusted_at`, timestamps, softDeletes
- Index: `(employee_id, year)`

### `audit_logs`
- `id`, `actor_id`, `action`, `entity_type`, `entity_id`
- `old_values` JSON, `new_values` JSON, `ip_address`, `user_agent`
- `trace_id`, `created_at`
- Index: `(entity_type, entity_id)`, `(actor_id)`, `(action)`, `(created_at)`

## 6) Modèles et relations (Eloquent)

- `User`:
  - `belongsTo(Role::class)`
  - `belongsTo(Service::class)`
  - `belongsTo(Department::class)`
  - `belongsTo(Position::class)`
  - `belongsTo(User::class, 'manager_id')`
  - `hasMany(User::class, 'manager_id')`
  - `hasMany(LeaveRequest::class, 'employee_id')`

- `LeaveRequest`:
  - `belongsTo(User::class, 'employee_id')`
  - `belongsTo(User::class, 'final_decision_by')`
  - `hasMany(LeaveApproval::class)`
  - casts: `start_date`, `end_date`, `status` (enum)

- `LeaveApproval`:
  - `belongsTo(LeaveRequest::class)`
  - `belongsTo(ApprovalLevel::class)`
  - `belongsTo(User::class, 'approver_id')`

- `LeaveBalance`:
  - `belongsTo(User::class, 'employee_id')`

- `AuditLog`:
  - `belongsTo(User::class, 'actor_id')`

## 7) Service métier principal (workflow + transaction)

```php
public function decide(LeaveRequest $request, User $actor, string $decision, ?string $comment = null): LeaveRequest
{
    return DB::transaction(function () use ($request, $actor, $decision, $comment) {
        $request->refresh();

        $this->authorizeTransition($request, $actor, $decision);

        $level = $this->resolveCurrentLevel($request->status);

        $approval = $this->leaveApprovalRepo->create([
            'leave_request_id' => $request->id,
            'approval_level_id' => $level->id,
            'approver_id' => $actor->id,
            'decision' => $decision,
            'comment' => $comment,
            'decided_at' => now(),
        ]);

        $newStatus = $this->computeNextStatus($request->status, $decision, $level->code);

        $request->update([
            'status' => $newStatus,
            'current_level_order' => $this->orderFromStatus($newStatus),
            'final_decision_by' => $newStatus->value === 'APPROVED' || $newStatus->value === 'REJECTED' ? $actor->id : null,
            'final_decision_at' => in_array($newStatus->value, ['APPROVED', 'REJECTED'], true) ? now() : null,
        ]);

        if ($newStatus->value === 'APPROVED') {
            $this->leaveBalanceService->deductDays($request->employee_id, $request->days_requested);
        }

        $this->auditService->logTransition($actor, $request, $approval, $newStatus);
        SendLeaveStatusNotificationJob::dispatch($request->id);

        return $request->fresh(['employee', 'approvals.approvalLevel', 'approvals.approver']);
    });
}
```

### Machine à états (transitions autorisées)
- `CREATED` -> `PENDING_SERVICE_HEAD`
- `PENDING_SERVICE_HEAD`
  - APPROVED -> `PENDING_HIERARCHY`
  - REJECTED -> `REJECTED`
- `PENDING_HIERARCHY`
  - APPROVED -> `PENDING_DGA_OPINION`
  - REJECTED -> `REJECTED`
- `PENDING_DGA_OPINION`
  - OPINION_* -> `PENDING_DG_OPINION`
- `PENDING_DG_OPINION`
  - OPINION_* -> `PENDING_HR_DECISION`
- `PENDING_HR_DECISION`
  - APPROVED -> `APPROVED`
  - REJECTED -> `REJECTED`

## 8) Policies Laravel (contrôle des transitions)

`LeaveRequestPolicy` doit vérifier:
- rôle actif de l’utilisateur
- état courant de la demande
- appartenance hiérarchique
- autorisation de niveau (service head, hierarchy, dga, dg, hr)
- interdiction de valider deux fois le même niveau
- blocage de toute action si état final (`APPROVED`/`REJECTED`)

## 9) Controllers et routes

### Contrôleurs
- `AuthController`: login/logout + rotation session
- `DashboardController`: KPIs par rôle
- `LeaveRequestController`: créer/modifier/lister/voir
- `LeaveApprovalController`: décisions/opinions
- `ExportController`: export PDF signé (id demande + hash)

### Routes (exemple)

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);

    Route::middleware('role:EMPLOYEE')->group(function () {
        Route::resource('leave-requests', LeaveRequestController::class)->only(['index', 'store', 'show', 'update']);
    });

    Route::middleware('role:SERVICE_HEAD,HIERARCHY,DGA,DG,HR')->group(function () {
        Route::post('/leave-requests/{leaveRequest}/approve', [LeaveApprovalController::class, 'approve']);
        Route::post('/leave-requests/{leaveRequest}/reject', [LeaveApprovalController::class, 'reject']);
        Route::post('/leave-requests/{leaveRequest}/opinion', [LeaveApprovalController::class, 'opinion']);
    });

    Route::middleware('role:HR,ADMIN')->group(function () {
        Route::get('/exports/leave-requests/{leaveRequest}/pdf', [ExportController::class, 'decisionPdf']);
    });
});
```

## 10) Visibilité employé (exigence explicite)

À **chaque événement** (création, validation, rejet, passage en attente) :
- notification in-app + email + websocket
- timeline visible dans `leave_requests.show`
- badge statut temps réel:
  - `EN ATTENTE` pour états `PENDING_*`
  - `VALIDÉ` pour `APPROVED`
  - `REJETÉ` pour `REJECTED`
- détail du niveau courant (ex: “En attente d’avis DG”)

## 11) Dashboard UI (moderne)

- Sidebar par rôle
- Cartes statistiques:
  - Mes congés
  - En attente
  - Approuvés
  - Rejetés
- Tableau filtrable: période, type, statut, service
- Composant “Workflow stepper” (étapes colorées)
- Centre de notifications avec accusé de lecture

## 12) Sécurité renforcée (standard banque)

- HTTPS only + HSTS + CSP stricte
- Mots de passe forts + MFA pour RH/Admin
- Sessions sécurisées (rotation ID, timeout court)
- Chiffrement des données sensibles (`encrypted` cast Laravel)
- Journal d’audit inviolable (write-only logique)
- RBAC strict + policies + middleware
- Rate limit sur login et endpoints sensibles
- WAF + IDS en frontal
- Sauvegardes chiffrées + PRA/PCA
- Ségrégation des environnements (dev/test/prod)
- Secret management (Vault / AWS Secrets Manager)

## 13) Scalabilité grande organisation

- Queue workers horizontaux (notifications, exports)
- Read replicas MySQL pour reporting
- Partitionnement audit_logs (mensuel)
- Indexation + EXPLAIN systématique
- Cache applicatif (dashboards, référentiels)
- Architecture modulaire par domaine
- Observabilité: logs centralisés, métriques Prometheus, traces OpenTelemetry

## 14) Tests unitaires et feature minimum

- `LeaveWorkflowTest`
  - refuse transition illégale
  - rejets Service Head / Hierarchy stoppent flux
  - opinions DGA/DG non bloquantes
  - seule RH finalise
  - mise à jour solde uniquement si APPROVED

- `LeaveStateTransitionTest`
  - matrice complète des transitions autorisées/interdites

## 15) Plan d’implémentation pratique

1. Initialiser projet Laravel + packages sécurité
2. Créer enums + migrations + seed rôles/niveaux
3. Implémenter modèles/relations/repositories
4. Implémenter workflow service transactionnel
5. Ajouter policies + middleware rôles
6. Brancher notifications queue + temps réel
7. Construire dashboards et stepper workflow
8. Mettre en place audit + monitoring + tests
9. Revue sécurité (pentest interne)
10. Déploiement progressif (blue/green)

---

## Exigence clé rappelée

Le système doit garantir qu’un employé voie **en permanence** l’état exact de sa demande (en attente, validée, rejetée), avec traçabilité complète des actions de validation dans Laravel et MySQL.
