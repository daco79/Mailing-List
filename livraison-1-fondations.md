# Application Mailing Lists Multi-Sociétés — Livraison 1/2

**Étapes couvertes** : 1. Architecture · 2. Schéma · 3. Migrations · 4. Modèles · 5. Relations · 6. Middleware de contexte

Stack : Laravel 12, MySQL 8, PHP 8.3+, Blade + Bootstrap 5, Queue database/redis.

---

## 1. Architecture générale multi-sociétés

### 1.1 Principes

1. **Une seule base MySQL**, multi-tenant logique via un `company_id` sur toutes les entités métier.
2. **Contacts mutualisés** : la table `contacts` stocke une seule fois chaque email. La table `company_contacts` porte le statut par société (actif, désabonné, consentement, source, type…).
3. **Isolation automatique** via un global scope Eloquent appliqué par un trait `BelongsToCompany`. Toute requête sur un modèle métier est filtrée par la société active sans effort du contrôleur.
4. **Société active** stockée dans la session utilisateur, résolue par un middleware (`SetCurrentCompany`) qui alimente un singleton `App\Services\CurrentCompany`.
5. **Permissions par société** via la table pivot `company_user` avec un champ `role` et un champ `permissions` JSON (souple pour l'avenir).
6. **Queues Laravel** pour les imports lourds et les envois massifs. Chaque société peut avoir son propre mailer résolu dynamiquement via un `MailerManager`.
7. **Migration future vers base séparée** : préparée en isolant tout accès société via `CurrentCompany` et en évitant les requêtes cross-tenant sauvages. Une société pourra être migrée en bougeant ses tables vers une autre connection.

### 1.2 Arborescence Laravel (parties concernées par la Livraison 1)

```
app/
├── Models/
│   ├── User.php
│   ├── Company.php
│   ├── Contact.php
│   ├── CompanyContact.php
│   ├── MailingList.php           # "list" est réservé en PHP
│   ├── Import.php
│   ├── Campaign.php
│   ├── CampaignRecipient.php
│   ├── SmtpSetting.php
│   ├── UnsubscribeLog.php
│   ├── GlobalBlacklistEmail.php
│   └── ActivityLog.php
├── Services/
│   └── CurrentCompany.php        # Singleton contexte société actif
├── Scopes/
│   └── CompanyScope.php          # Global scope auto-filter company_id
├── Traits/
│   └── BelongsToCompany.php      # Trait appliquant CompanyScope + auto-fill
├── Http/
│   └── Middleware/
│       ├── SetCurrentCompany.php     # Résout + alimente CurrentCompany
│       ├── EnsureCompanyAccess.php   # Vérifie l'accès user→company
│       └── EnsureIsAdmin.php         # (plus tard) super-admin
└── Providers/
    └── AppServiceProvider.php    # Binding du singleton

database/migrations/
├── 2026_04_23_000001_create_companies_table.php
├── 2026_04_23_000002_alter_users_table_add_admin_flag.php
├── 2026_04_23_000003_create_company_user_table.php
├── 2026_04_23_000004_create_contacts_table.php
├── 2026_04_23_000005_create_company_contacts_table.php
├── 2026_04_23_000006_create_lists_table.php
├── 2026_04_23_000007_create_list_company_contact_table.php
├── 2026_04_23_000008_create_imports_table.php
├── 2026_04_23_000009_create_campaigns_table.php
├── 2026_04_23_000010_create_campaign_recipients_table.php
├── 2026_04_23_000011_create_smtp_settings_table.php
├── 2026_04_23_000012_create_unsubscribe_logs_table.php
├── 2026_04_23_000013_create_global_blacklist_emails_table.php
└── 2026_04_23_000014_create_activity_logs_table.php
```

---

## 2. Schéma des tables (vue d'ensemble)

```
┌──────────────┐      ┌─────────────────┐      ┌──────────────┐
│   users      │◄────►│  company_user   │◄────►│  companies   │
└──────────────┘      │ (role, perms)   │      └──────┬───────┘
                      └─────────────────┘             │
                                                      ├──► smtp_settings  (1-1)
                                                      │
┌──────────────┐      ┌──────────────────┐            │
│  contacts    │◄────►│ company_contacts │◄───────────┤
│ (email uniq) │      │ (status, consent)│            │
└──────────────┘      └────────┬─────────┘            │
                               │                       │
                               ├──► list_company_contact ──► lists ─┤
                               │                                     │
                               └──► campaign_recipients ─► campaigns ┤
                                                                     │
                                          imports ────────────────── │
                                          unsubscribe_logs ───────── │
                                          activity_logs ──────────── │
                                                                     │
                              global_blacklist_emails  (pas de FK société)
```

---

## 3. Migrations (tous les fichiers)

### 3.1 `create_companies_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('companies', function (Blueprint $table) {
            $table->id();
            $table->string('name');                          // Nom d'usage / interne
            $table->string('legal_name')->nullable();        // Raison sociale
            $table->string('brand_name')->nullable();        // Marque commerciale
            $table->string('slug')->unique();                // Identifiant URL
            $table->string('default_from_name')->nullable();
            $table->string('default_from_email')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();

            $table->index('is_active');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('companies');
    }
};
```

### 3.2 `alter_users_table_add_admin_flag.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            // Super-admin global : accès à toutes les sociétés, gestion users
            $table->boolean('is_super_admin')->default(false)->after('password');
            $table->unsignedBigInteger('current_company_id')->nullable()->after('is_super_admin');
            // Pas de FK ici pour éviter les soucis d'ordre de migration ; on se fie à la logique app.
            $table->index('current_company_id');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropIndex(['current_company_id']);
            $table->dropColumn(['is_super_admin', 'current_company_id']);
        });
    }
};
```

### 3.3 `create_company_user_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('company_user', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->cascadeOnDelete();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            // Rôle lisible : owner | admin | operator | viewer
            $table->string('role')->default('operator');
            // Permissions fines (override du rôle) : JSON libre
            $table->json('permissions')->nullable();
            $table->timestamps();

            $table->unique(['company_id', 'user_id']);
            $table->index('role');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('company_user');
    }
};
```

### 3.4 `create_contacts_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('contacts', function (Blueprint $table) {
            $table->id();
            $table->string('first_name')->nullable();
            $table->string('last_name')->nullable();
            // Email unique GLOBAL : un seul enregistrement contact par email
            $table->string('email')->unique();
            $table->string('phone')->nullable();
            // "company_name" = nom de société indiqué dans le fichier source,
            // à ne pas confondre avec la table `companies` du tenant.
            $table->string('company_name')->nullable();
            $table->string('city')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->index('last_name');
            $table->index('city');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('contacts');
    }
};
```

### 3.5 `create_company_contacts_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('company_contacts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->restrictOnDelete();
            $table->foreignId('contact_id')->constrained()->restrictOnDelete();

            $table->string('source')->nullable();        // import, manual, form, api
            $table->string('contact_type')->nullable();  // prospect, client, lead, partner

            // active | unsubscribed | bounced | complained | deleted | suppressed
            $table->string('status')->default('active');

            // granted | pending | revoked | not_asked
            $table->string('consent_status')->default('not_asked');
            $table->timestamp('consent_date')->nullable();

            $table->timestamp('unsubscribe_date')->nullable();
            $table->string('unsubscribe_reason')->nullable();

            $table->text('notes')->nullable();

            $table->timestamps();
            $table->softDeletes();

            $table->unique(['company_id', 'contact_id']);
            $table->index(['company_id', 'status']);
            $table->index(['company_id', 'contact_type']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('company_contacts');
    }
};
```

### 3.6 `create_lists_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('lists', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->text('description')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->unique(['company_id', 'name']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('lists');
    }
};
```

### 3.7 `create_list_company_contact_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('list_company_contact', function (Blueprint $table) {
            $table->id();
            $table->foreignId('list_id')->constrained('lists')->cascadeOnDelete();
            $table->foreignId('company_contact_id')->constrained('company_contacts')->cascadeOnDelete();
            $table->timestamps();

            $table->unique(['list_id', 'company_contact_id'], 'list_company_contact_unique');
            $table->index('company_contact_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('list_company_contact');
    }
};
```

### 3.8 `create_imports_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('imports', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->cascadeOnDelete();
            $table->foreignId('user_id')->nullable()->constrained('users')->nullOnDelete(); // imported_by
            $table->string('filename');                 // Nom stocké sur le disque
            $table->string('original_filename');        // Nom d'origine uploadé
            $table->string('status')->default('pending'); // pending|processing|completed|failed
            $table->json('mapping')->nullable();        // Mapping colonnes → champs
            $table->json('options')->nullable();        // Options (liste cible, type par défaut…)

            $table->unsignedInteger('total_rows')->default(0);
            $table->unsignedInteger('inserted_rows')->default(0);
            $table->unsignedInteger('updated_rows')->default(0);
            $table->unsignedInteger('skipped_rows')->default(0);
            $table->unsignedInteger('error_rows')->default(0);

            $table->text('error_log')->nullable();
            $table->timestamp('started_at')->nullable();
            $table->timestamp('finished_at')->nullable();
            $table->timestamps();

            $table->index(['company_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('imports');
    }
};
```

### 3.9 `create_campaigns_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('campaigns', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->restrictOnDelete();
            $table->string('name');
            $table->string('subject');
            $table->string('sender_name');
            $table->string('sender_email');
            $table->string('reply_to_email')->nullable();
            $table->longText('html_content');
            $table->text('text_content')->nullable();

            // draft | scheduled | queued | sending | sent | paused | failed | canceled
            $table->string('status')->default('draft');

            $table->timestamp('scheduled_at')->nullable();
            $table->timestamp('sent_at')->nullable();

            $table->unsignedInteger('total_recipients')->default(0);
            $table->unsignedInteger('total_sent')->default(0);
            $table->unsignedInteger('total_failed')->default(0);
            $table->unsignedInteger('total_opened')->default(0);
            $table->unsignedInteger('total_clicked')->default(0);
            $table->unsignedInteger('total_unsubscribed')->default(0);

            $table->timestamps();
            $table->softDeletes();

            $table->index(['company_id', 'status']);
            $table->index('scheduled_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('campaigns');
    }
};
```

### 3.10 `create_campaign_recipients_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('campaign_recipients', function (Blueprint $table) {
            $table->id();
            $table->foreignId('campaign_id')->constrained()->cascadeOnDelete();
            // company_contact peut être supprimé ensuite ; on garde la trace ici.
            $table->foreignId('company_contact_id')->nullable()->constrained('company_contacts')->nullOnDelete();

            // Snapshot de l'email au moment de l'envoi (résistant aux changements ultérieurs)
            $table->string('email_snapshot');
            // Token unique pour la désinscription 1-clic
            $table->string('unsubscribe_token', 64)->unique();

            // pending | sent | failed | bounced | skipped
            $table->string('send_status')->default('pending');

            $table->timestamp('sent_at')->nullable();
            $table->timestamp('opened_at')->nullable();
            $table->timestamp('clicked_at')->nullable();
            $table->timestamp('unsubscribed_at')->nullable();

            $table->text('smtp_response')->nullable();
            $table->text('error_message')->nullable();
            $table->unsignedTinyInteger('retry_count')->default(0);

            $table->timestamps();

            $table->index(['campaign_id', 'send_status']);
            $table->index('email_snapshot');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('campaign_recipients');
    }
};
```

### 3.11 `create_smtp_settings_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('smtp_settings', function (Blueprint $table) {
            $table->id();
            // 1 configuration SMTP par société (unique)
            $table->foreignId('company_id')->unique()->constrained()->cascadeOnDelete();
            $table->string('host');
            $table->unsignedSmallInteger('port');
            $table->string('username');
            $table->text('password_encrypted'); // chiffré via Crypt::encryptString
            $table->string('encryption')->nullable(); // ssl | tls | null
            $table->string('from_email');
            $table->string('from_name');
            $table->string('reply_to_email')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamp('last_tested_at')->nullable();
            $table->string('last_test_result')->nullable(); // success | failure
            $table->text('last_test_error')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('smtp_settings');
    }
};
```

### 3.12 `create_unsubscribe_logs_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('unsubscribe_logs', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->cascadeOnDelete();
            $table->foreignId('company_contact_id')->nullable()->constrained('company_contacts')->nullOnDelete();
            $table->foreignId('campaign_id')->nullable()->constrained('campaigns')->nullOnDelete();

            $table->string('email');
            $table->string('token', 64)->nullable();
            $table->timestamp('unsubscribed_at');
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            // email_link | manual | admin | api | bounce | complaint
            $table->string('source')->default('email_link');

            $table->timestamps();

            $table->index(['company_id', 'email']);
            $table->index('token');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('unsubscribe_logs');
    }
};
```

### 3.13 `create_global_blacklist_emails_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('global_blacklist_emails', function (Blueprint $table) {
            $table->id();
            $table->string('email')->unique();
            $table->string('reason')->nullable();
            $table->foreignId('added_by')->nullable()->constrained('users')->nullOnDelete();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('global_blacklist_emails');
    }
};
```

### 3.14 `create_activity_logs_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('activity_logs', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->nullable()->constrained()->nullOnDelete();
            $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();
            $table->string('action');                           // company.created, import.started...
            $table->string('entity_type')->nullable();          // App\Models\Campaign
            $table->unsignedBigInteger('entity_id')->nullable();
            $table->json('details_json')->nullable();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            // Log append-only : pas d'updated_at
            $table->timestamp('created_at')->useCurrent();

            $table->index(['company_id', 'created_at']);
            $table->index(['entity_type', 'entity_id']);
            $table->index('action');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('activity_logs');
    }
};
```

---

## 4. Modèles Eloquent

### 4.1 `app/Services/CurrentCompany.php`

```php
<?php

namespace App\Services;

use App\Models\Company;

/**
 * Singleton résolvant la société active dans le contexte courant.
 * Alimenté par le middleware SetCurrentCompany.
 * Utilisé par le global scope CompanyScope.
 */
class CurrentCompany
{
    protected ?Company $company = null;

    public function set(?Company $company): void
    {
        $this->company = $company;
    }

    public function get(): ?Company
    {
        return $this->company;
    }

    public function id(): ?int
    {
        return $this->company?->id;
    }

    public function isSet(): bool
    {
        return $this->company !== null;
    }

    public function clear(): void
    {
        $this->company = null;
    }
}
```

### 4.2 `app/Scopes/CompanyScope.php`

```php
<?php

namespace App\Scopes;

use App\Services\CurrentCompany;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

/**
 * Applique automatiquement un WHERE company_id = current_company
 * sur tout modèle utilisant le trait BelongsToCompany.
 *
 * Peut être désactivé ponctuellement via ->withoutGlobalScope(CompanyScope::class).
 */
class CompanyScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        /** @var CurrentCompany $current */
        $current = app(CurrentCompany::class);

        if (! $current->isSet()) {
            // Aucune société active : on ne filtre rien ici mais on refuse
            // l'accès au niveau middleware. En CLI ou jobs, on utilise explicitement
            // ->withoutGlobalScope(CompanyScope::class) ou on set() la société.
            return;
        }

        $builder->where($model->qualifyColumn('company_id'), $current->id());
    }
}
```

### 4.3 `app/Traits/BelongsToCompany.php`

```php
<?php

namespace App\Traits;

use App\Models\Company;
use App\Scopes\CompanyScope;
use App\Services\CurrentCompany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

/**
 * À appliquer sur tous les modèles possédant une colonne company_id
 * qui doit être filtrée automatiquement par société active.
 */
trait BelongsToCompany
{
    public static function bootBelongsToCompany(): void
    {
        // Global scope de lecture
        static::addGlobalScope(new CompanyScope());

        // Auto-remplissage de company_id à la création si manquant
        static::creating(function ($model) {
            if (empty($model->company_id)) {
                $current = app(CurrentCompany::class);
                if ($current->isSet()) {
                    $model->company_id = $current->id();
                }
            }
        });
    }

    public function company(): BelongsTo
    {
        return $this->belongsTo(Company::class);
    }
}
```

### 4.4 `app/Models/User.php`

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    protected $fillable = [
        'name', 'email', 'password',
        'is_super_admin', 'current_company_id',
    ];

    protected $hidden = [
        'password', 'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password'          => 'hashed',
            'is_super_admin'    => 'boolean',
        ];
    }

    public function companies(): BelongsToMany
    {
        return $this->belongsToMany(Company::class)
            ->withPivot(['role', 'permissions'])
            ->withTimestamps();
    }

    public function currentCompany(): BelongsTo
    {
        return $this->belongsTo(Company::class, 'current_company_id');
    }

    /**
     * L'utilisateur a-t-il accès à cette société ?
     */
    public function canAccessCompany(Company $company): bool
    {
        if ($this->is_super_admin) {
            return true;
        }
        return $this->companies()->whereKey($company->id)->exists();
    }

    /**
     * Rôle de l'utilisateur pour une société donnée, ou null.
     */
    public function roleFor(Company $company): ?string
    {
        if ($this->is_super_admin) {
            return 'super_admin';
        }
        $pivot = $this->companies()->whereKey($company->id)->first()?->pivot;
        return $pivot?->role;
    }

    /**
     * Permission ponctuelle pour la société active.
     */
    public function hasPermissionFor(Company $company, string $permission): bool
    {
        if ($this->is_super_admin) {
            return true;
        }
        $pivot = $this->companies()->whereKey($company->id)->first()?->pivot;
        if (! $pivot) {
            return false;
        }
        // Role owner/admin = tout autorisé par convention
        if (in_array($pivot->role, ['owner', 'admin'], true)) {
            return true;
        }
        $perms = is_string($pivot->permissions) ? json_decode($pivot->permissions, true) : ($pivot->permissions ?? []);
        return in_array($permission, $perms ?? [], true);
    }
}
```

### 4.5 `app/Models/Company.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Str;

class Company extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'name', 'legal_name', 'brand_name', 'slug',
        'default_from_name', 'default_from_email',
        'is_active',
    ];

    protected function casts(): array
    {
        return [
            'is_active' => 'boolean',
        ];
    }

    protected static function booted(): void
    {
        static::creating(function (Company $company) {
            if (empty($company->slug)) {
                $company->slug = static::generateUniqueSlug($company->name);
            }
        });
    }

    protected static function generateUniqueSlug(string $name): string
    {
        $base = Str::slug($name);
        $slug = $base ?: 'company';
        $i    = 1;
        while (static::where('slug', $slug)->exists()) {
            $slug = $base . '-' . $i++;
        }
        return $slug;
    }

    /* ---- Relations ---- */

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class)
            ->withPivot(['role', 'permissions'])
            ->withTimestamps();
    }

    public function companyContacts(): HasMany
    {
        return $this->hasMany(CompanyContact::class);
    }

    public function lists(): HasMany
    {
        return $this->hasMany(MailingList::class);
    }

    public function campaigns(): HasMany
    {
        return $this->hasMany(Campaign::class);
    }

    public function imports(): HasMany
    {
        return $this->hasMany(Import::class);
    }

    public function smtpSetting(): HasOne
    {
        return $this->hasOne(SmtpSetting::class);
    }

    public function unsubscribeLogs(): HasMany
    {
        return $this->hasMany(UnsubscribeLog::class);
    }

    public function activityLogs(): HasMany
    {
        return $this->hasMany(ActivityLog::class);
    }
}
```

### 4.6 `app/Models/Contact.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

/**
 * Contact "global" : informations neutres, indépendantes d'une société.
 * Un même email = un seul contact, partagé entre sociétés via company_contacts.
 *
 * ATTENTION : pas de BelongsToCompany ici, c'est une entité globale.
 */
class Contact extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'first_name', 'last_name', 'email',
        'phone', 'company_name', 'city',
    ];

    protected static function booted(): void
    {
        static::saving(function (Contact $c) {
            if (! empty($c->email)) {
                $c->email = strtolower(trim($c->email));
            }
        });
    }

    public function companyContacts(): HasMany
    {
        return $this->hasMany(CompanyContact::class);
    }

    public function fullName(): string
    {
        return trim(($this->first_name ?? '') . ' ' . ($this->last_name ?? ''));
    }
}
```

### 4.7 `app/Models/CompanyContact.php`

```php
<?php

namespace App\Models;

use App\Traits\BelongsToCompany;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class CompanyContact extends Model
{
    use HasFactory, SoftDeletes, BelongsToCompany;

    public const STATUS_ACTIVE       = 'active';
    public const STATUS_UNSUBSCRIBED = 'unsubscribed';
    public const STATUS_BOUNCED      = 'bounced';
    public const STATUS_COMPLAINED   = 'complained';
    public const STATUS_SUPPRESSED   = 'suppressed';

    public const CONSENT_GRANTED   = 'granted';
    public const CONSENT_PENDING   = 'pending';
    public const CONSENT_REVOKED   = 'revoked';
    public const CONSENT_NOT_ASKED = 'not_asked';

    protected $fillable = [
        'company_id', 'contact_id',
        'source', 'contact_type',
        'status', 'consent_status',
        'consent_date', 'unsubscribe_date', 'unsubscribe_reason',
        'notes',
    ];

    protected function casts(): array
    {
        return [
            'consent_date'     => 'datetime',
            'unsubscribe_date' => 'datetime',
        ];
    }

    /* ---- Relations ---- */

    public function contact(): BelongsTo
    {
        return $this->belongsTo(Contact::class);
    }

    public function lists(): BelongsToMany
    {
        return $this->belongsToMany(MailingList::class, 'list_company_contact', 'company_contact_id', 'list_id')
            ->withTimestamps();
    }

    public function campaignRecipients(): HasMany
    {
        return $this->hasMany(CampaignRecipient::class);
    }

    /* ---- Helpers ---- */

    public function isActive(): bool
    {
        return $this->status === self::STATUS_ACTIVE;
    }

    public function isUnsubscribed(): bool
    {
        return $this->status === self::STATUS_UNSUBSCRIBED;
    }

    public function markUnsubscribed(?string $reason = null): void
    {
        $this->forceFill([
            'status'             => self::STATUS_UNSUBSCRIBED,
            'consent_status'     => self::CONSENT_REVOKED,
            'unsubscribe_date'   => now(),
            'unsubscribe_reason' => $reason,
        ])->save();
    }

    /* ---- Scopes ---- */

    public function scopeActive($q)
    {
        return $q->where('status', self::STATUS_ACTIVE);
    }

    public function scopeSendable($q)
    {
        return $q->whereIn('status', [self::STATUS_ACTIVE])
                 ->whereIn('consent_status', [self::CONSENT_GRANTED, self::CONSENT_NOT_ASKED]);
    }
}
```

### 4.8 `app/Models/MailingList.php`

```php
<?php

namespace App\Models;

use App\Traits\BelongsToCompany;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;

/**
 * La table SQL s'appelle "lists".
 * On nomme le modèle MailingList parce que "List" est un mot-clé réservé en PHP 7+.
 */
class MailingList extends Model
{
    use HasFactory, SoftDeletes, BelongsToCompany;

    protected $table = 'lists';

    protected $fillable = [
        'company_id', 'name', 'description',
    ];

    public function companyContacts(): BelongsToMany
    {
        return $this->belongsToMany(
            CompanyContact::class,
            'list_company_contact',
            'list_id',
            'company_contact_id'
        )->withTimestamps();
    }
}
```

### 4.9 `app/Models/Import.php`

```php
<?php

namespace App\Models;

use App\Traits\BelongsToCompany;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Import extends Model
{
    use HasFactory, BelongsToCompany;

    public const STATUS_PENDING    = 'pending';
    public const STATUS_PROCESSING = 'processing';
    public const STATUS_COMPLETED  = 'completed';
    public const STATUS_FAILED     = 'failed';

    protected $fillable = [
        'company_id', 'user_id',
        'filename', 'original_filename',
        'status', 'mapping', 'options',
        'total_rows', 'inserted_rows', 'updated_rows', 'skipped_rows', 'error_rows',
        'error_log', 'started_at', 'finished_at',
    ];

    protected function casts(): array
    {
        return [
            'mapping'     => 'array',
            'options'     => 'array',
            'started_at'  => 'datetime',
            'finished_at' => 'datetime',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

### 4.10 `app/Models/Campaign.php`

```php
<?php

namespace App\Models;

use App\Traits\BelongsToCompany;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Campaign extends Model
{
    use HasFactory, SoftDeletes, BelongsToCompany;

    public const STATUS_DRAFT     = 'draft';
    public const STATUS_SCHEDULED = 'scheduled';
    public const STATUS_QUEUED    = 'queued';
    public const STATUS_SENDING   = 'sending';
    public const STATUS_SENT      = 'sent';
    public const STATUS_PAUSED    = 'paused';
    public const STATUS_FAILED    = 'failed';
    public const STATUS_CANCELED  = 'canceled';

    protected $fillable = [
        'company_id', 'name', 'subject',
        'sender_name', 'sender_email', 'reply_to_email',
        'html_content', 'text_content',
        'status', 'scheduled_at', 'sent_at',
        'total_recipients', 'total_sent', 'total_failed',
        'total_opened', 'total_clicked', 'total_unsubscribed',
    ];

    protected function casts(): array
    {
        return [
            'scheduled_at' => 'datetime',
            'sent_at'      => 'datetime',
        ];
    }

    public function recipients(): HasMany
    {
        return $this->hasMany(CampaignRecipient::class);
    }

    public function isEditable(): bool
    {
        return in_array($this->status, [self::STATUS_DRAFT, self::STATUS_SCHEDULED], true);
    }
}
```

### 4.11 `app/Models/CampaignRecipient.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Support\Str;

/**
 * Pas de BelongsToCompany : la société est inférée via campaign_id.
 * Pour les besoins de filtrage admin, on joindra campaigns.company_id.
 */
class CampaignRecipient extends Model
{
    use HasFactory;

    public const STATUS_PENDING = 'pending';
    public const STATUS_SENT    = 'sent';
    public const STATUS_FAILED  = 'failed';
    public const STATUS_BOUNCED = 'bounced';
    public const STATUS_SKIPPED = 'skipped';

    protected $fillable = [
        'campaign_id', 'company_contact_id',
        'email_snapshot', 'unsubscribe_token',
        'send_status', 'sent_at', 'opened_at', 'clicked_at', 'unsubscribed_at',
        'smtp_response', 'error_message', 'retry_count',
    ];

    protected function casts(): array
    {
        return [
            'sent_at'         => 'datetime',
            'opened_at'       => 'datetime',
            'clicked_at'      => 'datetime',
            'unsubscribed_at' => 'datetime',
        ];
    }

    protected static function booted(): void
    {
        static::creating(function (CampaignRecipient $r) {
            if (empty($r->unsubscribe_token)) {
                $r->unsubscribe_token = Str::random(48);
            }
        });
    }

    public function campaign(): BelongsTo
    {
        return $this->belongsTo(Campaign::class);
    }

    public function companyContact(): BelongsTo
    {
        return $this->belongsTo(CompanyContact::class);
    }
}
```

### 4.12 `app/Models/SmtpSetting.php`

```php
<?php

namespace App\Models;

use App\Traits\BelongsToCompany;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Crypt;

class SmtpSetting extends Model
{
    use HasFactory, BelongsToCompany;

    protected $fillable = [
        'company_id',
        'host', 'port', 'username', 'password_encrypted',
        'encryption', 'from_email', 'from_name', 'reply_to_email',
        'is_active', 'last_tested_at', 'last_test_result', 'last_test_error',
    ];

    protected $hidden = ['password_encrypted'];

    protected function casts(): array
    {
        return [
            'is_active'      => 'boolean',
            'last_tested_at' => 'datetime',
        ];
    }

    /**
     * Setter : on chiffre le mot de passe via Laravel Crypt (clé APP_KEY).
     */
    public function setPasswordAttribute(?string $value): void
    {
        $this->attributes['password_encrypted'] = $value !== null && $value !== ''
            ? Crypt::encryptString($value)
            : $this->attributes['password_encrypted'] ?? null;
    }

    /**
     * Accesseur transparent pour l'utilisation dans le mailer.
     */
    public function getPasswordAttribute(): ?string
    {
        if (empty($this->password_encrypted)) {
            return null;
        }
        try {
            return Crypt::decryptString($this->password_encrypted);
        } catch (\Throwable $e) {
            return null;
        }
    }
}
```

### 4.13 `app/Models/UnsubscribeLog.php`

```php
<?php

namespace App\Models;

use App\Traits\BelongsToCompany;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class UnsubscribeLog extends Model
{
    use HasFactory, BelongsToCompany;

    public const SOURCE_EMAIL_LINK = 'email_link';
    public const SOURCE_MANUAL     = 'manual';
    public const SOURCE_ADMIN      = 'admin';
    public const SOURCE_API        = 'api';
    public const SOURCE_BOUNCE     = 'bounce';
    public const SOURCE_COMPLAINT  = 'complaint';

    protected $fillable = [
        'company_id', 'company_contact_id', 'campaign_id',
        'email', 'token', 'unsubscribed_at',
        'ip_address', 'user_agent', 'source',
    ];

    protected function casts(): array
    {
        return [
            'unsubscribed_at' => 'datetime',
        ];
    }

    public function companyContact(): BelongsTo
    {
        return $this->belongsTo(CompanyContact::class);
    }

    public function campaign(): BelongsTo
    {
        return $this->belongsTo(Campaign::class);
    }
}
```

### 4.14 `app/Models/GlobalBlacklistEmail.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

/**
 * Blacklist GLOBALE, pas filtrée par société : utilisée comme garde-fou
 * pour bloquer les envois vers certaines adresses quelle que soit la société.
 */
class GlobalBlacklistEmail extends Model
{
    use HasFactory;

    protected $fillable = ['email', 'reason', 'added_by'];

    protected static function booted(): void
    {
        static::saving(function (GlobalBlacklistEmail $b) {
            if (! empty($b->email)) {
                $b->email = strtolower(trim($b->email));
            }
        });
    }

    public function addedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'added_by');
    }

    public static function isBlacklisted(string $email): bool
    {
        return static::where('email', strtolower(trim($email)))->exists();
    }
}
```

### 4.15 `app/Models/ActivityLog.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class ActivityLog extends Model
{
    use HasFactory;

    public $timestamps = false;

    protected $fillable = [
        'company_id', 'user_id', 'action',
        'entity_type', 'entity_id', 'details_json',
        'ip_address', 'user_agent', 'created_at',
    ];

    protected function casts(): array
    {
        return [
            'details_json' => 'array',
            'created_at'   => 'datetime',
        ];
    }

    public function company(): BelongsTo
    {
        return $this->belongsTo(Company::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Helper statique pour tracer une action rapidement.
     */
    public static function record(
        string $action,
        ?int $companyId = null,
        ?int $userId = null,
        ?string $entityType = null,
        ?int $entityId = null,
        array $details = []
    ): self {
        $request = request();
        return static::create([
            'company_id'   => $companyId,
            'user_id'      => $userId ?? auth()->id(),
            'action'       => $action,
            'entity_type'  => $entityType,
            'entity_id'    => $entityId,
            'details_json' => $details,
            'ip_address'   => $request?->ip(),
            'user_agent'   => $request?->userAgent(),
            'created_at'   => now(),
        ]);
    }
}
```

---

## 5. Relations Eloquent (résumé)

| Depuis               | Type            | Vers              | Détail                                                      |
|----------------------|-----------------|-------------------|-------------------------------------------------------------|
| User                 | belongsToMany   | Company           | via `company_user` avec pivot `role`, `permissions`         |
| User                 | belongsTo       | Company           | `current_company_id` (société active stockée)               |
| Company              | belongsToMany   | User              | inverse                                                     |
| Company              | hasOne          | SmtpSetting       | 1 SMTP par société                                          |
| Company              | hasMany         | CompanyContact    |                                                             |
| Company              | hasMany         | MailingList       |                                                             |
| Company              | hasMany         | Campaign          |                                                             |
| Company              | hasMany         | Import            |                                                             |
| Company              | hasMany         | UnsubscribeLog    |                                                             |
| Company              | hasMany         | ActivityLog       |                                                             |
| Contact              | hasMany         | CompanyContact    | Un contact global peut être dans plusieurs sociétés         |
| CompanyContact       | belongsTo       | Company           |                                                             |
| CompanyContact       | belongsTo       | Contact           |                                                             |
| CompanyContact       | belongsToMany   | MailingList       | via `list_company_contact`                                  |
| CompanyContact       | hasMany         | CampaignRecipient |                                                             |
| MailingList          | belongsTo       | Company           |                                                             |
| MailingList          | belongsToMany   | CompanyContact    |                                                             |
| Campaign             | belongsTo       | Company           |                                                             |
| Campaign             | hasMany         | CampaignRecipient |                                                             |
| CampaignRecipient    | belongsTo       | Campaign          |                                                             |
| CampaignRecipient    | belongsTo       | CompanyContact    |                                                             |
| SmtpSetting          | belongsTo       | Company           |                                                             |
| UnsubscribeLog       | belongsTo       | Company           |                                                             |
| UnsubscribeLog       | belongsTo       | CompanyContact    |                                                             |
| UnsubscribeLog       | belongsTo       | Campaign          |                                                             |
| ActivityLog          | belongsTo       | Company / User    |                                                             |
| GlobalBlacklistEmail | — (global)      | —                 | Pas de rattachement société                                 |

---

## 6. Middleware de contexte société

### 6.1 `app/Http/Middleware/SetCurrentCompany.php`

```php
<?php

namespace App\Http\Middleware;

use App\Models\Company;
use App\Services\CurrentCompany;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

/**
 * Résout la société active pour l'utilisateur authentifié et la pousse
 * dans le singleton CurrentCompany ainsi que dans la session.
 *
 * Priorités de résolution :
 *   1. `?company=<id|slug>` dans l'URL (switch explicite)
 *   2. session('current_company_id')
 *   3. user.current_company_id
 *   4. 1re société accessible par l'utilisateur
 */
class SetCurrentCompany
{
    public function __construct(private CurrentCompany $current) {}

    public function handle(Request $request, Closure $next): Response
    {
        $user = $request->user();

        if (! $user) {
            return $next($request);
        }

        $company = $this->resolveCompany($request, $user);

        if ($company) {
            // Vérifie l'accès, sauf super-admin
            if (! $user->canAccessCompany($company)) {
                abort(403, "Vous n'avez pas accès à cette société.");
            }

            $this->current->set($company);
            session(['current_company_id' => $company->id]);

            // Persiste aussi côté user pour la prochaine session
            if ($user->current_company_id !== $company->id) {
                $user->forceFill(['current_company_id' => $company->id])->save();
            }
        }

        return $next($request);
    }

    protected function resolveCompany(Request $request, $user): ?Company
    {
        // 1. Switch explicite via ?company=
        if ($request->filled('company')) {
            $value = $request->query('company');
            $company = is_numeric($value)
                ? Company::find($value)
                : Company::where('slug', $value)->first();
            if ($company && $user->canAccessCompany($company)) {
                return $company;
            }
        }

        // 2. Session
        if ($id = session('current_company_id')) {
            $company = Company::find($id);
            if ($company && $user->canAccessCompany($company)) {
                return $company;
            }
        }

        // 3. User
        if ($user->current_company_id) {
            $company = Company::find($user->current_company_id);
            if ($company && $user->canAccessCompany($company)) {
                return $company;
            }
        }

        // 4. Première société accessible
        if ($user->is_super_admin) {
            return Company::where('is_active', true)->orderBy('id')->first();
        }
        return $user->companies()->where('is_active', true)->orderBy('companies.id')->first();
    }
}
```

### 6.2 `app/Http/Middleware/EnsureCompanyAccess.php`

```php
<?php

namespace App\Http\Middleware;

use App\Services\CurrentCompany;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

/**
 * À placer sur toutes les routes métier.
 * Refuse l'accès si aucune société n'est active dans le contexte courant.
 */
class EnsureCompanyAccess
{
    public function __construct(private CurrentCompany $current) {}

    public function handle(Request $request, Closure $next): Response
    {
        if (! $this->current->isSet()) {
            // L'utilisateur n'a encore rattaché aucune société ou n'a pas pu être résolu.
            return redirect()
                ->route('companies.select')
                ->with('warning', 'Veuillez sélectionner une société pour continuer.');
        }

        if (! $this->current->get()->is_active) {
            abort(403, 'Cette société est désactivée.');
        }

        return $next($request);
    }
}
```

### 6.3 `app/Http/Middleware/EnsureIsSuperAdmin.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureIsSuperAdmin
{
    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->user() || ! $request->user()->is_super_admin) {
            abort(403, 'Accès réservé aux super-administrateurs.');
        }
        return $next($request);
    }
}
```

### 6.4 `app/Providers/AppServiceProvider.php` (extrait)

```php
<?php

namespace App\Providers;

use App\Services\CurrentCompany;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Singleton contexte société, utilisé par le scope et le middleware.
        $this->app->singleton(CurrentCompany::class, fn () => new CurrentCompany());
    }

    public function boot(): void
    {
        //
    }
}
```

### 6.5 Enregistrement des middlewares (`bootstrap/app.php`, Laravel 11/12)

```php
<?php

use App\Http\Middleware\EnsureCompanyAccess;
use App\Http\Middleware\EnsureIsSuperAdmin;
use App\Http\Middleware\SetCurrentCompany;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web:      __DIR__ . '/../routes/web.php',
        commands: __DIR__ . '/../routes/console.php',
        health:   '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Alias utilisables dans les routes.
        $middleware->alias([
            'company'        => SetCurrentCompany::class,
            'company.access' => EnsureCompanyAccess::class,
            'super_admin'    => EnsureIsSuperAdmin::class,
        ]);

        // SetCurrentCompany doit tourner APRÈS l'auth mais AVANT les contrôleurs.
        // On l'ajoute par défaut au groupe web pour toutes les routes authentifiées.
        $middleware->web(append: [
            SetCurrentCompany::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })
    ->create();
```

### 6.6 Utilisation dans les routes (aperçu, détaillé en Livraison 2)

```php
Route::middleware(['auth', 'company.access'])->group(function () {
    Route::get('/dashboard',    [DashboardController::class, 'index'])->name('dashboard');
    Route::resource('contacts', ContactController::class);
    Route::resource('lists',    ListController::class);
    Route::resource('campaigns',CampaignController::class);
    // ... etc. Toutes ces routes bénéficient de l'isolation automatique.
});
```

---

## Installation & premières commandes

```bash
# 1. Créer le projet
composer create-project laravel/laravel mailing-admin
cd mailing-admin

# 2. Configurer la base MySQL dans .env (DB_DATABASE, DB_USERNAME, DB_PASSWORD)
# 3. Configurer la queue : QUEUE_CONNECTION=database (ou redis)
php artisan queue:table && php artisan migrate

# 4. Placer les fichiers de migration, modèles, services, scopes, traits, middlewares
#    tels que décrits ci-dessus.
# 5. Lancer les migrations
php artisan migrate

# 6. Vérifier que le singleton est bien résolu
php artisan tinker
>>> app(App\Services\CurrentCompany::class)
```

---

## Notes importantes sur l'isolation

1. **Jobs et CLI** : dans un job de queue ou une commande artisan, il n'y a pas de user authentifié → `CurrentCompany` n'est pas set. Il faudra l'alimenter explicitement au début du job :

   ```php
   app(CurrentCompany::class)->set($company);
   ```

   Sinon, bypass explicite : `Model::withoutGlobalScope(CompanyScope::class)->…`.

2. **Requêtes cross-tenant** (rares : super-admin listant tous les imports) : utiliser explicitement `withoutGlobalScope(CompanyScope::class)`.

3. **Tests** : dans les tests, set la société active avant toute assertion sur des modèles filtrés.

4. **Migration future vers base séparée** : puisque tous les accès aux entités métier passent par `CurrentCompany` + scope, migrer une société reviendra à :
   - créer une nouvelle connexion dans `config/database.php`,
   - dupliquer ses tables sur cette connexion,
   - router ses modèles via `getConnectionName()` dynamique en fonction de `CurrentCompany`.

---

## Prochaine livraison (Livraison 2/2)

Je livrerai dans le message suivant, dans le même niveau de détail :

7. **Routes** (web + public désinscription)
8. **Contrôleurs** (Auth, Companies, Contacts, CompanyContacts, Lists, Imports, Campaigns, Smtp, Unsubscribe, Logs)
9. **Vues Blade** (layout admin Bootstrap, index/create/edit/show pour chaque ressource, page publique de désinscription)
10. **Logique d'import** : `ImportWizardController` + `ProcessImportJob` (chunked, doublons, préservation du statut désabonné)
11. **Logique d'envoi** : `SendCampaignJob` + `SendCampaignEmailJob`, résolution dynamique du mailer via `MailerManager` (SMTP par société), exclusion blacklist + désabonnés + emails invalides, enregistrement `campaign_recipients`
12. **Désinscription** : route publique `GET /u/{token}`, journal, mise à jour statut société-spécifique
13. **Isolation & sécurité** : policies, gates, tests d'isolation, confirmations avant suppressions dangereuses

Dis-moi si tu veux que j'ajuste quoi que ce soit dans la Livraison 1 avant d'attaquer la 2 (par exemple : une convention de nommage, des index supplémentaires, utilisation de ULID au lieu de bigint, ou un découpage différent des statuts).
