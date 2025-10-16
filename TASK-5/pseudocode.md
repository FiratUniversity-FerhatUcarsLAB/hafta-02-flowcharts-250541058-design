############################################
# Data structures
############################################

User:
    id
    name
    roles            // e.g., member, staff, admin
    card_id_hash     // hash of card chip ID (store hashed)
    fingerprint_template_hash  // hash of stored fingerprint template (not raw image)
    fingerprint_enrolled = false
    card_enrolled = false
    access_enabled = true
    lock_until = null  // timestamp if temporarily locked due to suspicious activity
    last_failed_attempts = 0
    last_failed_timestamp = null

AuditLogEntry:
    timestamp
    user_id_or_card_id_or_anonymous
    event_type        // ENROLL, ENTRY_SUCCESS, ENTRY_FAIL, SPOOF_DETECTED, OVERRIDE, etc.
    device_id
    message
    signature         // signed by server/HSM to prevent tampering

SystemConfig:
    MAX_FAILED_ATTEMPTS = 5
    LOCK_DURATION = 15 minutes
    SPOOF_CONFIDENCE_THRESHOLD = 0.7  // threshold for anti-spoof classifier
    MATCH_THRESHOLD = 0.6             // fingerprint template match threshold
    OTP_TTL = 5 minutes
    AUDIT_SIGNING_KEY  // stored in HSM
    ENROLL_FLOWS_ALLOW_GUARD_OVERRIDE = true

Device:
    id
    location
    has_card_reader = true/false
    has_fingerprint_sensor = true/false
    has_liveness_sensors = {thermal, capacitive, pulse, micro_movement}
    tamper_sensor = boolean

############################################
# Helper / primitive functions (assumed secure)
############################################

function secure_hash(data) -> string
    // cryptographic hash (e.g., SHA-256 with salt stored per-user)
    ...

function sign_log(entry: AuditLogEntry) -> signature
    // sign with HSM key
    ...

function store_audit(entry: AuditLogEntry)
    entry.signature = sign_log(entry)
    append_to_audit_store(entry)

function current_time() -> timestamp
    ...

function generate_otp(user) -> otp
    // generate one-time code, store with ttl
    ...

function send_otp_to_user(user, otp)
    // SMS / email through verified channel
    ...

function alert_security_team(message)
    // push notification / SMS / call to on-call guard
    ...

############################################
# Enrollment flows
############################################

function enroll_card(user, card_chip_id):
    if not authorized_operator():
        return error("Not authorized to enroll")
    hashed = secure_hash(card_chip_id + user.id_salt)
    user.card_id_hash = hashed
    user.card_enrolled = true
    store_audit(AuditLogEntry(current_time(), user.id, "ENROLL_CARD", device_id=current_device(), message="Card enrolled for user"))
    return success

function enroll_fingerprint(user, fingerprint_raw_samples[]):
    if not authorized_operator():
        return error("Not authorized to enroll")
    # Collect multiple samples to build a robust template
    template = build_fingerprint_template(fingerprint_raw_samples)  # runs inside device/HSM
    # Derive template hash (do not store raw template in clear)
    template_hash = secure_hash(template + user.id_salt)
    user.fingerprint_template_hash = template_hash
    user.fingerprint_enrolled = true
    store_secure_template(user.id, template)     # store template in HSM/secure enclave only
    store_audit(AuditLogEntry(current_time(), user.id, "ENROLL_FINGERPRINT", device_id=current_device(), message="Fingerprint enrolled"))
    return success

############################################
# Liveness / Anti-spoofing function
############################################

function check_liveness(sensor_data) -> (is_live: bool, confidence: float):
    # Combine multiple signals: thermal pattern, capacitive read, pulse/oximetry, micro-movement, material reflectance
    # Each sub-detector returns a score 0..1; aggregate via weighted model
    thermal_score = thermal_detector(sensor_data.thermal)
    capacitive_score = capacitive_detector(sensor_data.capacitance)
    pulse_score = pulse_detector(sensor_data.pulse_waveform)
    micro_move_score = micro_movement_detector(sensor_data.micro_motion)
    material_score = material_reflectance_detector(sensor_data.optical)
    combined_score = weighted_average([thermal_score, capacitive_score, pulse_score, micro_move_score, material_score])
    is_live = combined_score >= SystemConfig.SPOOF_CONFIDENCE_THRESHOLD
    return (is_live, combined_score)

############################################
# Fingerprint match (runs securely inside HSM/device)
############################################

function match_fingerprint(probe_sample, user) -> (match: bool, score: float):
    stored_template = retrieve_secure_template(user.id)  // from HSM, not readable externally
    score = compute_template_similarity(probe_sample, stored_template)
    return (score >= SystemConfig.MATCH_THRESHOLD, score)

############################################
# Card verification
############################################

function verify_card(chip_id, user) -> bool:
    hashed = secure_hash(chip_id + user.id_salt)
    return hashed == user.card_id_hash

############################################
# Access control checks
############################################

function check_user_allowed(user, action) -> (allowed: bool, reason: string):
    if user is null:
        return (false, "Unknown user")
    if not user.access_enabled:
        return (false, "Access disabled")
    if user.lock_until != null and current_time() < user.lock_until:
        return (false, "Temporarily locked")
    // Additional rules (membership expired, unpaid fees, banned)
    if membership_expired(user):
        return (false, "Membership expired")
    return (true, "")

############################################
# Main entry authentication flow
############################################

function authenticate_entry(device: Device, input):
    # input may contain: card_chip_id (optional), fingerprint_probe (optional), operator_override = false
    # Step 0: device tamper check
    if device.tamper_sensor_triggered():
        store_audit(AuditLogEntry(current_time(), "unknown", "DEVICE_TAMPER", device.id, "Tamper detected"))
        alert_security_team("Tamper on device " + device.id)
        deny_access_with_lockdown()
        return failure("Device tampered")

    # Step 1: identify candidate user(s) from card if present
    candidate_user = null
    if input.card_chip_id present:
        candidate_user = lookup_user_by_card_hash( secure_hash(input.card_chip_id + known_salt) )
        if candidate_user == null:
            # unknown card - log and possible soft-fail
            store_audit(AuditLogEntry(current_time(), input.card_chip_id, "CARD_UNKNOWN", device.id, "Unknown card attempted"))
            # proceed to fingerprint-only flow or ask guard
    # Step 2: fingerprint path (preferred)
    if input.fingerprint_probe present:
        # Anti-spoof / liveness check first
        (is_live, live_conf) = check_liveness(input.sensor_data)
        if not is_live:
            # suspicious -> increment counters, log, possible lock
            store_audit(AuditLogEntry(current_time(), candidate_user?.id or "unknown", "SPOOF_DETECTED", device.id, "Liveness failed, score="+live_conf))
            handle_suspicious_attempt(candidate_user, device, "liveness_fail")
            return failure("Liveness check failed")
        # Next, try to match with identified user (if card known) or search database (1:N) with limits
        if candidate_user != null:
            (matched, score) = match_fingerprint(input.fingerprint_probe, candidate_user)
            if matched:
                return finalize_entry(candidate_user, device, method="fingerprint", score=score, live_conf=live_conf)
            else:
                # Might be wrong finger or cloned template; treat as failure
                store_audit(AuditLogEntry(current_time(), candidate_user.id, "FINGERPRINT_MISMATCH", device.id, "Score="+score))
                handle_failed_attempt(candidate_user, device)
                return failure("Fingerprint mismatch")
        else:
            # No card: perform 1:N search but with rate/latency limits and inside HSM
            match_result = search_fingerprint_templates_secure(input.fingerprint_probe, max_results=1, time_limit=2s)
            if match_result.found:
                user = match_result.user
                return finalize_entry(user, device, method="fingerprint", score=match_result.score, live_conf=live_conf)
            else:
                store_audit(AuditLogEntry(current_time(), "unknown", "FINGERPRINT_NO_MATCH", device.id, "No matching template"))
                handle_failed_attempt(null, device)
                return failure("No matching fingerprint")
    # Step 3: card-only fallback
    if input.card_chip_id present:
        user = lookup_user_by_card_hash(secure_hash(input.card_chip_id + known_salt))
        if user != null:
            allowed, reason = check_user_allowed(user, "entry")
            if not allowed:
                store_audit(AuditLogEntry(current_time(), user.id, "ACCESS_DENIED", device.id, reason))
                return failure(reason)
            if verify_card(input.card_chip_id, user):
                # optionally require PIN / OTP for higher security
                if user_requires_pin(user):
                    if not verify_pin(input.pin, user):
                        handle_failed_attempt(user, device)
                        return failure("PIN incorrect")
                return finalize_entry(user, device, method="card")
            else:
                store_audit(AuditLogEntry(current_time(), user.id, "CARD_VERIFICATION_FAIL", device.id, "Card hash mismatch"))
                handle_failed_attempt(user, device)
                return failure("Card verification failed")
    # Step 4: Operator/guard override (last resort)
    if input.operator_override_requested and authorized_guard_present():
        if SystemConfig.ENROLL_FLOWS_ALLOW_GUARD_OVERRIDE:
            guard_id = current_operator()
            store_audit(AuditLogEntry(current_time(), guard_id, "OVERRIDE_GRANTED", device.id, "Guard override for unknown user"))
            allow_entry_with_manual_check()
            return success("Entry allowed by guard override")
    # final fallback: deny
    store_audit(AuditLogEntry(current_
