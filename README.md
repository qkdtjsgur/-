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
    public Vector2 rollColliderSize = new Vector2(0.8f, 0.6f);        // 구르기 중 Collider 크기

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

    void Start()
    {
        rb = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        col = GetComponent<BoxCollider2D>();
    }

    void Update()
    {
        // 구르기 중에는 다른 입력 무시
        if (isRolling)
        {
            rollTimer -= Time.deltaTime;
            if (rollTimer <= 0f)
                EndRoll();
            return;
        }

        // 이동
        moveInput = Input.GetAxisRaw("Horizontal");
        rb.linearVelocity = new Vector2(moveInput * moveSpeed, rb.linearVelocity.y);

        // 애니메이션 파라미터 갱신
        animator.SetFloat("speed", Mathf.Abs(moveInput));
        animator.SetBool("isGrounded", isGrounded);
        animator.SetBool("isRolling", isRolling);

        // 점프
        if (Input.GetKeyDown(KeyCode.Space) && isGrounded)
        {
            Jump();
        }

        // 구르기
        if (Input.GetKeyDown(KeyCode.LeftShift) && isGrounded)
        {
            StartRoll();
        }

        // 좌우 반전
        if (moveInput != 0)
            transform.localScale = new Vector3(Mathf.Sign(moveInput), 1, 1);
    }

    void Jump()
    {
        rb.linearVelocity = new Vector2(rb.linearVelocity.x, jumpForce);
        animator.SetTrigger("jump");
    }

    void StartRoll()
    {
        isRolling = true;
        // Collider 크기 줄이기
        col.size = rollColliderSize;
        animator.SetBool("isRolling", true);
        animator.SetTrigger("roll");
        rollTimer = rollDuration;


        // 구르는 방향으로 이동
        float rollDirection = Mathf.Sign(transform.localScale.x);
        rb.linearVelocity = new Vector2(rollDirection * rollSpeed, rb.linearVelocity.y);

      
    }

    void EndRoll()
    {
        isRolling = false;
        animator.SetBool("isRolling", false);

        // Collider 복원
        col.size = normalColliderSize;
    }

    void FixedUpdate()
    {
        isGrounded = Physics2D.OverlapCircle(groundCheck.position, 0.12f, groundLayer);
    }
}
