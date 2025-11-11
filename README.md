using UnityEngine;

public class PlayerMovement : MonoBehaviour
{
    [Header("이동 설정")]
    public float moveSpeed = 5f;
    public float jumpForce = 7f;

    [Header("구르기 설정")]
    public float rollSpeed = 10f;                 // 구르기 속도
    public float rollDuration = 0.5f;             // 구르기 지속시간
    public Vector2 normalColliderSize = new Vector2(0.8355505f, 1.268564f); // 현재 Collider 크기
    public Vector2 rollColliderSize = new Vector2(0.5f, 0.3f);        // 구르기 중 Collider 크기

    public bool isRolling = false;
    public float rollTimer;

    [Header("땅 감지 설정")]
    public Transform groundCheck;
    public LayerMask groundLayer;

    private Rigidbody2D rb;
    private Animator animator;
    private BoxCollider2D col;
    private bool isGrounded;
    private float moveInput;

    // 내부 저장용 (원상복구)
    private Vector2 savedColliderSize;
    private Vector2 savedColliderOffset;

    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        col = GetComponent<BoxCollider2D>();

        // 현재 콜라이더 사이즈/오프셋 저장 (나중에 복원)
        if (col != null)
        {
            savedColliderSize = col.size;
            savedColliderOffset = col.offset;
            // 초기값이 스크립트에 적혀있는 normalColliderSize와 다르면 동기화
            normalColliderSize = savedColliderSize;
        }
    }

    void Update()
    {
        // --- 최신 지면 판정 (Update 초반에 계산해서 바로 입력에 반영) ---
        if (groundCheck != null)
            isGrounded = Physics2D.OverlapCircle(groundCheck.position, 0.12f, groundLayer);
        else
            isGrounded = false;

        animator.SetBool("isGrounded", isGrounded);

        // 구르기 중에는 다른 입력 무시(타이머만 감소)
        if (isRolling)
        {
            rollTimer -= Time.deltaTime;
            if (rollTimer <= 0f)
                EndRoll();
            return;
        }

        // 이동
        moveInput = Input.GetAxisRaw("Horizontal");
        // rb.linearVelocity 대신 rb.velocity 사용 (안정성)
        rb.linearVelocity = new Vector2(moveInput * moveSpeed, rb.linearVelocity.y);

        // 애니메이션 파라미터 갱신
        if (animator != null)
        {
            animator.SetFloat("speed", Mathf.Abs(moveInput));
            animator.SetBool("isRolling", isRolling);
        }

        // 좌우 반전 (localScale 방식 유지)
        if (moveInput != 0)
            transform.localScale = new Vector3(Mathf.Sign(moveInput), 1, 1);

        // 점프 (지면에 있을 때만)
        if (Input.GetKeyDown(KeyCode.Space) && isGrounded)
        {
            Jump();
        }

        // 구르기 (LeftShift) — 공중 구르기 방지: 수직 속도 거의 0 인지 확인
        if (Input.GetKeyDown(KeyCode.LeftShift))
        {
            bool verticalStill = Mathf.Abs(rb.linearVelocity.y) < 0.05f; // 작게 잡아서 공중 감지 여유치 확보
            if (!isRolling && isGrounded && verticalStill)
            {
                StartRoll();
            }
        }
    }

    void Jump()
    {
        // 점프할 때 수직속도 세팅
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
        if (animator != null) animator.SetTrigger("jump");
    }

    void StartRoll()
    {
        isRolling = true;
        rollTimer = rollDuration;

        // 콜라이더 크기 줄이기 (안정성을 위해 saved 값 바탕으로 적용)
        if (col != null)
        {
            col.size = rollColliderSize;
            // offset은 고정 유지(원하면 조정 가능)
            // col.offset = new Vector2(savedColliderOffset.x, savedColliderOffset.y - 0.2f);
        }

        // 애니메이터 갱신
        if (animator != null)
        {
            animator.SetBool("isRolling", true);
            // 만약 Animator가 Trigger "roll"을 사용한다면 아래 줄도 활성화
            animator.SetTrigger("roll");
        }

        // 구르는 방향으로 빠르게 이동
        float rollDirection = Mathf.Sign(transform.localScale.x);
        rb.linearVelocity = new Vector2(rollDirection * rollSpeed, rb.linearVelocity.y);
    }

    void EndRoll()
    {
        isRolling = false;

        // 콜라이더 원복
        if (col != null)
        {
            col.size = savedColliderSize; // normalColliderSize 로도 가능
            col.offset = savedColliderOffset;
        }

        if (animator != null)
            animator.SetBool("isRolling", false);
    }

    void FixedUpdate()
    {
        // 기존 방식처럼 FixedUpdate에서 지면 체크 원하면 유지
        // (여기서는 Update 쪽에서 이미 isGrounded를 갱신했으므로 굳이 다시 해도 무방)
        // isGrounded = Physics2D.OverlapCircle(groundCheck.position, 0.12f, groundLayer);
    }

    // 시각적 확인용 Gizmo
    private void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = Color.yellow;
            Gizmos.DrawWireSphere(groundCheck.position, 0.12f);
        }
    }
}
