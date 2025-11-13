using UnityEngine;

public class PlayerMovement : MonoBehaviour
{
    [Header("이동 설정")]
    public float moveSpeed = 5f;
    public float jumpForce = 7f;

    [Header("구르기 설정")]
    public float rollSpeed = 10f;
    public float rollDuration = 0.5f;
    public Vector2 normalColliderSize = new Vector2(0.8f, 1.2f);
    public Vector2 rollColliderSize = new Vector2(0.4f, 0.6f);

    public bool isRolling = false;
    public float rollTimer;

    [Header("공격 설정")]
    public float comboResetTime = 1.0f;   // 콤보 리셋 시간
    private int comboStep = 0;            // 현재 콤보 단계 (1~3)
    private float lastAttackTime;         // 마지막 공격 시점
    private bool isAttacking = false;

    [Header("땅 감지 설정")]
    public Transform groundCheck;
    public LayerMask groundLayer;

    private Rigidbody2D rb;
    private Animator animator;
    private BoxCollider2D col;
    private bool isGrounded;
    private float moveInput;

    private Vector2 savedColliderSize;
    private Vector2 savedColliderOffset;

    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        col = GetComponent<BoxCollider2D>();

        rollColliderSize = new Vector2(0.6f, 0.52f);

        if (col != null)
        {
            savedColliderSize = col.size;
            savedColliderOffset = col.offset;
            normalColliderSize = savedColliderSize;
        }
    }

    void Update()
    {
        // --- 지면 판정 ---
        if (groundCheck != null)
            isGrounded = Physics2D.OverlapCircle(groundCheck.position, 0.12f, groundLayer);

        animator.SetBool("isGrounded", isGrounded);

        // --- 구르기 중이면 다른 입력 무시 ---
        if (isRolling)
        {
            rollTimer -= Time.deltaTime;
            if (rollTimer <= 0f)
                EndRoll();
            return;
        }

        // --- 공격 중 ---
        if (isAttacking)
        {
            // 이동 정지
            rb.linearVelocity = new Vector2(0, rb.linearVelocity.y);
            // 콤보 타이머 경과 시 공격 해제
            if (Time.time - lastAttackTime > 0.5f)
                isAttacking = false;
            return;
        }

        // --- 이동 ---
        moveInput = Input.GetAxisRaw("Horizontal");
        rb.linearVelocity = new Vector2(moveInput * moveSpeed, rb.linearVelocity.y);

        animator.SetFloat("speed", Mathf.Abs(moveInput));
        animator.SetBool("isRolling", isRolling);

        if (moveInput != 0)
            transform.localScale = new Vector3(Mathf.Sign(moveInput), 1, 1);

        // --- 점프 ---
        if (Input.GetKeyDown(KeyCode.Space) && isGrounded)
            Jump();

        // --- 구르기 ---
        if (Input.GetKeyDown(KeyCode.LeftShift))
        {
            bool verticalStill = Mathf.Abs(rb.linearVelocity.y) < 0.05f;
            if (!isRolling && isGrounded && verticalStill)
                StartRoll();
        }

        // --- 공격 (마우스 왼쪽 클릭) ---
        if (Input.GetMouseButtonDown(0))
        {
            TryAttack();
        }
    }

    void Jump()
    {
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
        animator.SetTrigger("jump");
    }

    void StartRoll()
    {
        isRolling = true;
        rollTimer = rollDuration;

        col.size = rollColliderSize;

        animator.SetBool("isRolling", true);
        animator.SetTrigger("roll");

        float rollDirection = Mathf.Sign(transform.localScale.x);
        rb.linearVelocity = new Vector2(rollDirection * rollSpeed, rb.linearVelocity.y);
    }

    void EndRoll()
    {
        isRolling = false;
        col.size = savedColliderSize;
        col.offset = savedColliderOffset;
        animator.SetBool("isRolling", false);
    }

    // 🥊 공격 로직
    void TryAttack()
    {
        // 조건: 낙하 중, 구르기 중엔 공격 불가
        if (!isGrounded || isRolling) return;

        // 콤보 시간 초과 시 리셋
        if (Time.time - lastAttackTime > comboResetTime)
            comboStep = 0;

        comboStep++;
        if (comboStep > 3) comboStep = 1;

        // 공격 시작
        isAttacking = true;
        lastAttackTime = Time.time;

        // 이동 정지
        rb.linearVelocity = Vector2.zero;

        // 애니메이션 트리거
        switch (comboStep)
        {
            case 1:
                animator.SetTrigger("attack1");
                break;
            case 2:
                animator.SetTrigger("attack2");
                break;
            case 3:
                animator.SetTrigger("attack3");
                break;
        }

        // 일정 시간 후 공격 해제
        Invoke(nameof(EndAttack), 0.4f);
    }

    void EndAttack()
    {
        isAttacking = false;
    }

    private void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = Color.yellow;
            Gizmos.DrawWireSphere(groundCheck.position, 0.12f);
        }
    }
}
