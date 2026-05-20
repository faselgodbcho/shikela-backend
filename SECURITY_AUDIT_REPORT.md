# Shikela Backend Security Audit Report

Date: 2026-05-20 UTC

## Executive Summary
- Overall security score: **38/100**
- Findings: **10** (Critical 4, High 4, Medium 2)
- Assessment type: static code architecture and AppSec review across Django/DRF backend.

## Top Critical Issues
- Hardcoded cryptographic secrets in `core/settings.py`.
- IDOR on user detail endpoint in `account/views.py`.
- Missing tenant/object ownership checks across shop/catalog/inventory CRUD.
- Missing explicit authentication on key order endpoints.

## Detailed Findings
### SHK-001 - Hardcoded Django secret key and SantimPay private key in source
- Severity: **Critical**
- Affected: `core/settings.py`
- Vulnerable snippet:
```python
SECRET_KEY = "django-insecure-..."
SANTIMPAY_PRIVATE_KEY = _normalize_pem(os.getenv(..., '''-----BEGIN EC PRIVATE KEY-----...'''))
```
- Why vulnerable: Cryptographic secrets are committed in source control, enabling key theft and token/signature forgery.
- Real-world impact: Account/session compromise, payment signature forgery, and total environment compromise if reused across envs.
- Exploitation scenario: An attacker with repo read access uses leaked private key to sign malicious SantimPay requests and impersonate trusted payment operations.
- Recommended fix: Move all secrets to environment/secret manager; fail fast if missing; rotate leaked keys immediately.
- Example secure implementation:
```python
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
SANTIMPAY_PRIVATE_KEY = _normalize_pem(os.environ['SANTIMPAY_PRIVATE_KEY'])
```

### SHK-002 - Insecure production defaults (DEBUG=True, ALLOWED_HOSTS empty, missing secure headers)
- Severity: **High**
- Affected: `core/settings.py`
- Vulnerable snippet:
```python
DEBUG = True
ALLOWED_HOSTS = []
```
- Why vulnerable: Debug mode and weak host/origin hardening expose stack traces and weaken host header protections.
- Real-world impact: Information disclosure, easier exploit development, cache poisoning/host-header abuse in misconfigured deployments.
- Exploitation scenario: A production deployment forgets env overrides; attacker triggers errors to harvest secrets/config and internal paths.
- Recommended fix: Default to secure production posture with DEBUG=False, strict ALLOWED_HOSTS/CSRF_TRUSTED_ORIGINS/CORS allowlist and HSTS/security headers.
- Example secure implementation:
```python
DEBUG = os.getenv('DEBUG','false').lower()=='true'
if not DEBUG:
    SECURE_HSTS_SECONDS=31536000
    SESSION_COOKIE_SECURE=True
    CSRF_COOKIE_SECURE=True
```

### SHK-003 - Broken object-level authorization on user profile endpoint
- Severity: **Critical**
- Affected: `account/views.py`
- Vulnerable snippet:
```python
class UserDetailView(RetrieveUpdateDestroyAPIView):
    queryset = User.objects.all()
```
- Why vulnerable: No object-level permission checks ensure users can access only their own record.
- Real-world impact: IDOR allows viewing/updating/deleting arbitrary user accounts.
- Exploitation scenario: Authenticated user requests /auth/user/<other-user-uuid>/ and edits victim profile or deletes account.
- Recommended fix: Restrict queryset to request.user or enforce IsSelfOrAdmin permission class.
- Example secure implementation:
```python
def get_queryset(self):
    return User.objects.filter(id=self.request.user.id)
```

### SHK-004 - Authentication missing on critical order endpoints
- Severity: **Critical**
- Affected: `order/views.py`
- Vulnerable snippet:
```python
class BuyNowView(APIView):
    def post(...):
class CheckoutCartView(APIView):
    def post(...):
class ListOrdersView(APIView):
    def get(...):
```
- Why vulnerable: Views omit permission_classes, so API defaults may be bypassed/misapplied and business logic uses request.user directly.
- Real-world impact: Unauthorized order creation, cart checkout misuse, and data exposure if unauthenticated requests pass through with AnonymousUser edge cases.
- Exploitation scenario: Attacker hits /order/create/ without token to stress order creation path and probe logic flaws; with middleware drift this becomes full auth bypass.
- Recommended fix: Explicitly set permission_classes=[IsAuthenticated] and validate authenticated user type before processing.
- Example secure implementation:
```python
class BuyNowView(APIView):
    permission_classes=[permissions.IsAuthenticated]
```

### SHK-005 - Global data exposure due to missing ownership filters in CRUD endpoints
- Severity: **Critical**
- Affected: `shop/views.py, inventory/views.py, catalog/views.py`
- Vulnerable snippet:
```python
queryset = Shop.objects.all()
queryset = Inventory.objects...all()
queryset = Product.objects.all()
```
- Why vulnerable: List/detail/update/delete endpoints use unrestricted querysets with only IsAuthenticated, enabling cross-tenant read/write.
- Real-world impact: Multi-tenant isolation break: vendors can read/modify other vendors' shops, inventory, and products.
- Exploitation scenario: Shop owner updates another shop by ID through ShopDetailView PATCH.
- Recommended fix: Filter by tenant ownership in get_queryset and enforce per-object permission checks.
- Example secure implementation:
```python
def get_queryset(self):
    return Shop.objects.filter(owner=self.request.user)
```

### SHK-006 - Unvalidated user-supplied notify_url in payment initiation (SSRF/webhook hijack)
- Severity: **High**
- Affected: `payment/views.py, payment/services/service.py`
- Vulnerable snippet:
```python
notify_url = request.data.get('notify_url')
... service.direct_payment(... notify_url=notify_url ...)
```
- Why vulnerable: User-controlled callback URL is forwarded to payment provider without allowlisting.
- Real-world impact: SSRF-like callback abuse, exfiltration of transaction events to attacker-controlled endpoints, reconciliation tampering.
- Exploitation scenario: Attacker sets notify_url to malicious endpoint and receives sensitive payment lifecycle callbacks.
- Recommended fix: Do not accept notify_url from client; use server-side configured callback or strict allowlist.
- Example secure implementation:
```python
notify_url = settings.SANTIMPAY_NOTIFY_URL
```

### SHK-007 - Unsafe file upload controls missing (type/content scanning and size limits)
- Severity: **High**
- Affected: `account/models.py, catalog/models.py, shop/models.py, hub/models.py`
- Vulnerable snippet:
```python
FileField(upload_to='licenses/')
FileField(upload_to='products/media/')
ImageField(upload_to='...')
```
- Why vulnerable: No validators for extension/MIME/size or malware scanning pipeline are enforced.
- Real-world impact: Malicious file upload, storage abuse, possible XSS/content-sniffing issues when served, and legal/security risk.
- Exploitation scenario: Attacker uploads polyglot HTML/SVG payload as product media and shares link for stored-XSS in weakly configured file serving.
- Recommended fix: Add strict file validators, max size, antivirus scanning, and safe content-disposition in media serving.
- Example secure implementation:
```python
file = models.FileField(upload_to='products/media/', validators=[FileExtensionValidator(['jpg','png','pdf']), validate_file_size])
```

### SHK-008 - No API throttling/rate-limiting on auth and transactional endpoints
- Severity: **High**
- Affected: `core/settings.py, account/views.py, payment/views.py, order/views.py`
- Vulnerable snippet:
```python
REST_FRAMEWORK = { ... }  # no DEFAULT_THROTTLE_*
```
- Why vulnerable: Missing throttle classes enables brute-force/login abuse and high-volume transaction abuse.
- Real-world impact: Credential stuffing, OTP/email abuse, DoS and fraud automation.
- Exploitation scenario: Botnet sends massive login and payment-init requests without per-IP/user limits.
- Recommended fix: Enable DRF throttles plus edge rate-limits (Nginx/WAF/API gateway).
- Example secure implementation:
```python
REST_FRAMEWORK['DEFAULT_THROTTLE_CLASSES']=['rest_framework.throttling.UserRateThrottle','rest_framework.throttling.AnonRateThrottle']
```

### SHK-009 - Dependency/runtime misconfiguration risk: Django 6.0 pin likely incompatible
- Severity: **Medium**
- Affected: `requirements.txt`
- Vulnerable snippet:
```python
Django==6.0.2
```
- Why vulnerable: Pinned version may be unreleased/incompatible with project assumptions; also requirements file appears UTF-16 encoded, harming tooling/scanners.
- Real-world impact: Broken deployments, missed vulnerability scanning, undefined runtime behavior.
- Exploitation scenario: CI security scanner skips malformed requirements and misses vulnerable packages.
- Recommended fix: Use UTF-8 requirements, pin verified compatible versions, and run pip-audit/safety in CI.
- Example secure implementation:
```python
Django==5.1.*  # or tested LTS
# file encoded UTF-8
```

### SHK-010 - Payment/refund business-logic race windows need stronger idempotency and locking
- Severity: **Medium**
- Affected: `payment/views.py, payment/services/service.py`
- Vulnerable snippet:
```python
Refund.objects.create(... status=REQUESTED ...)
update_or_create(order=order, user=request.user, provider='SANTIMPAY', ...)
```
- Why vulnerable: Concurrent requests can create duplicate operational states without idempotency keys and select_for_update on critical rows.
- Real-world impact: Double-processing, inconsistent payout/refund accounting, financial reconciliation issues.
- Exploitation scenario: Client retries quickly during timeout; two refund requests pass validation before status updates settle.
- Recommended fix: Introduce idempotency keys, unique constraints for active requests, and row-level locking around financial transitions.
- Example secure implementation:
```python
with transaction.atomic():
    payment = Payment.objects.select_for_update().get(id=payment_id)
```

## Remediation Roadmap
1. **Immediate (24-48h):** rotate all leaked keys, disable DEBUG in production, add strict ALLOWED_HOSTS, lock down user endpoint authorization.
2. **Short term (1 week):** enforce ownership filters/object permissions in all multi-tenant resources; add auth/throttling on all order/payment endpoints.
3. **Medium term (2-4 weeks):** harden upload pipeline (validators + malware scan), implement payment idempotency keys and locking.
4. **Ongoing:** dependency governance (pip-audit in CI), SAST/DAST and security regression tests.

## Deployment Hardening Checklist
- [ ] DEBUG=False in production
- [ ] SECRET_KEY + payment keys in secret manager only
- [ ] Enforce HTTPS/HSTS/secure cookies
- [ ] Add CSRF trusted origins and CORS allowlist
- [ ] Enable DRF throttles and WAF/API gateway rate limits
- [ ] Restrict media serving domain + safe content types
- [ ] Enable centralized audit logging and alerting
- [ ] Add backup/restore and incident response runbooks