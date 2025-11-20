# Ex.No: 10  Implementation of 3D game

### DATE:                                                                            
### REGISTER NUMBER : 212223230252

## AIM: 
To develop a 3D game in Unity.
## Algorithm:
### 1. Environment Setup

1. Create a lava plane that serves as the danger zone.
2. Place multiple rock chunks scattered above the lava to act as safe jumping points.
3. Add a blue platform at the far end to act as the Finish goal.
4. Add a Player character with Rigidbody and appropriate Collider.

### 2. Layer & Tag Assignment
1. Assign all rock objects to a Rock Layer.
2. Tag the lava plane as "Lava".
3. Tag the blue finish platform as "Finish".
4. Keep the player tagged as "Player".

### 3. Player Input Handling

1. Read WASD/Arrow-key input each frame to determine a movement direction (h, v).
2. If movement keys are pressed → update the inputDir vector.
3. Optionally rotate a direction-arrow UI element above the player to show input direction.

### 4. Jump Mechanics

1. When the Space key is pressed, confirm the player is grounded:
   - Use a CheckSphere below the player.
   - Only allow jumping when grounded is true.

2. If grounded:

   - Apply upward force for the jump height.
   - Apply horizontal force based on inputDir toward the next rock.

### 5. Mid-Air Movement

1. While the player is in the air:
   - Continue reading directional input.
   - Adjust horizontal movement using controlled air steering.
   - Limit horizontal velocity to a maximum air speed.

2. This enables the player to correct jumps while airborne, including diagonal adjustments.

### 6. Collision & Trigger Detection

1. If the player contacts any object tagged “Lava”:
   - Trigger a Game Over event.
   - Disable all player movement controls.

2. If the player reaches the object tagged “Finish”:
   - Display the victory message.
   - Disable further player input.

### 7. UI Feedback

1. Use a TextMeshPro UI element for player messages.
2. Display:
   - “GAME OVER – You fell into the lava!” when touching lava
   - “YOU WIN – Reached the finish platform!” upon success

### 8. Invisible Platform Adjustment (Optional)

1. Add invisible colliders slightly above the rock surfaces to simplify landings.
2. Remove rendering so they act as unseen helpers.
3. The player lands on these invisible surfaces, giving smoother gameplay.

### 9. Continuous Game Loop

1. Game listens for player inputs during play.
2. Player moves, jumps, and steers between platforms.
3. System continuously checks:
   - Contact with lava → Game Over
   - Reaching finish platform → Victory

4. Game ends after either a win or a loss.


### Program:
#### PlayerController.cs
```c++
using UnityEngine;
using TMPro;

[RequireComponent(typeof(Rigidbody))]
public class PlayerController : MonoBehaviour
{
    [Header("Jump Settings")]
    public float jumpUpVelocity = 6f;
    public float jumpForwardSpeed = 8f;   // initial horizontal impulse on jump
    [Tooltip("How strongly the player can steer in mid-air (0..1 is reasonable). Higher = snappier.")]
    public float airControl = 0.8f;       // multiplier for velocity correction in air
    [Tooltip("Maximum horizontal speed allowed while in air")]
    public float maxAirSpeed = 6f;

    [Header("Ground Check")]
    public Transform groundCheck;
    public float groundCheckRadius = 0.25f;
    public LayerMask groundLayer;

    [Header("UI / Arrow Indicator")]
    public RectTransform arrowIndicatorUI;
    public float arrowRotateSpeed = 720f;

    [Header("TMP Message (optional)")]
    public TMP_Text messageText;

    // internals
    Rigidbody rb;
    Vector2 inputDir = new Vector2(0f, 1f); // current input direction (h, v)
    bool grounded;

    void Awake()
    {
        rb = GetComponent<Rigidbody>();
        rb.constraints = RigidbodyConstraints.FreezeRotation;
    }

    void Start()
    {
        if (messageText != null) messageText.text = "";

        if (groundCheck == null)
        {
            GameObject gc = new GameObject("GroundCheck");
            gc.transform.SetParent(transform);
            gc.transform.localPosition = new Vector3(0f, -0.6f, 0f);
            groundCheck = gc.transform;
        }
    }

// --- Debugging helpers - paste into PlayerController class temporarily ---

// ---------- Paste this Update() method (replace your current Update) ----------
void Update()
{
    // ------- Debug: occasional logs to diagnose input & ground check (non-spammy) -------
    if (Time.frameCount % 30 == 0) // roughly twice per second @60fps
    {
        bool sphere = Physics.CheckSphere(groundCheck.position, groundCheckRadius, groundLayer, QueryTriggerInteraction.Ignore);
        Debug.Log($"[DBG] frame={Time.frameCount} grounded={grounded} CheckSphere={sphere} groundPos={groundCheck.position} radius={groundCheckRadius}");
    }

    // Detect Space presses for debugging (still call jump normally below)
    if (Input.GetKeyDown(KeyCode.Space))
    {
        Debug.Log("[DBG] Space pressed (Update). grounded=" + grounded);
    }

    // -------- Input reading (robust: axis + direct key fallback) ----------
    float h = 0f, v = 0f;

    // Try axis first (works with Input Manager)
    h = Input.GetAxisRaw("Horizontal");
    v = Input.GetAxisRaw("Vertical");

    // If axes are effectively zero, fallback to explicit key checks so arrow keys always respond
    if (Mathf.Abs(h) < 0.01f && Mathf.Abs(v) < 0.01f)
    {
        if (Input.GetKey(KeyCode.LeftArrow) || Input.GetKey(KeyCode.A)) h = -1f;
        if (Input.GetKey(KeyCode.RightArrow) || Input.GetKey(KeyCode.D)) h =  1f;
        if (Input.GetKey(KeyCode.UpArrow) || Input.GetKey(KeyCode.W))    v =  1f;
        if (Input.GetKey(KeyCode.DownArrow) || Input.GetKey(KeyCode.S))  v = -1f;
    }

    // Always update inputDir each frame so it affects mid-air control
    if (Mathf.Abs(h) > 0.001f || Mathf.Abs(v) > 0.001f)
    {
        inputDir = new Vector2(h, v).normalized;
    }
    // (Optionally: else inputDir stays the last aimed direction)

    // Update visual arrow indicator
    UpdateArrowIndicator(inputDir);

    // Normal jump: only when grounded
    if (Input.GetKeyDown(KeyCode.Space) && grounded)
    {
        DoJump();
    }

    // Test force-jump key for debugging; press P to force DoJump regardless of grounded
    TestForceJump();
}

// ---------- Temporary helper to force a jump with P. Remove after debugging ----------
void TestForceJump()
{
    if (Input.GetKeyDown(KeyCode.P))
    {
        Debug.Log("[DBG] Force jump (P) triggered.");
        DoJump();
    }
}


    void FixedUpdate()
    {
        // update grounded state
        grounded = Physics.CheckSphere(groundCheck.position, groundCheckRadius, groundLayer, QueryTriggerInteraction.Ignore);

        // Mid-air control: we smoothly push current horizontal velocity toward target horizontal velocity
        // Compute current horizontal velocity
        Vector3 currentH = new Vector3(rb.linearVelocity.x, 0f, rb.linearVelocity.z);

        // Desired target horizontal based on current input direction
        Vector3 targetDir = new Vector3(inputDir.x, 0f, inputDir.y);
        Vector3 targetH = Vector3.zero;
        if (targetDir.sqrMagnitude > 0.001f)
        {
            targetH = targetDir.normalized * maxAirSpeed;
        }

        if (!grounded)
        {
            // Compute delta needed to reach targetH
            Vector3 delta = (targetH - currentH);

            // Scale delta by airControl and apply as an immediate velocity change (smoothness depends on airControl)
            // Clamp the applied delta so control feels reasonable
            Vector3 apply = delta * Mathf.Clamp01(airControl);

            rb.AddForce(apply, ForceMode.VelocityChange);
        }
        else
        {
            // Optional: small ground friction control to prevent sliding when standing
            // You can tune Rigidbody.drag in inspector instead of handling here.
        }
    }

    void UpdateArrowIndicator(Vector2 dir)
    {
        if (arrowIndicatorUI == null) return;

        float angle = Mathf.Atan2(dir.x, dir.y) * Mathf.Rad2Deg;
        float currentZ = arrowIndicatorUI.eulerAngles.z;
        float targetZ = -angle;
        float newZ = Mathf.MoveTowardsAngle(currentZ, targetZ, arrowRotateSpeed * Time.deltaTime);
        arrowIndicatorUI.eulerAngles = new Vector3(0f, 0f, newZ);
    }

    void DoJump()
    {
        // initial horizontal impulse on jump uses the input direction
        Vector3 horizontal = new Vector3(inputDir.x, 0f, inputDir.y).normalized;
        if (horizontal.magnitude < 0.01f) horizontal = transform.forward;

        // Reset vertical velocity for consistent jumps
        Vector3 vel = rb.linearVelocity;
        vel.y = 0f;
        rb.linearVelocity = vel;

        // Apply impulse: use forward speed and vertical
        Vector3 impulse = horizontal * jumpForwardSpeed;
        impulse.y = jumpUpVelocity;

        rb.AddForce(impulse, ForceMode.VelocityChange);
    }

    void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Water"))
        {
            if (GameManager.Instance != null) GameManager.Instance.GameOver();
            else if (messageText != null) messageText.text = "Game Over";
        }
        else if (other.CompareTag("Finish"))
        {
            if (GameManager.Instance != null) GameManager.Instance.Win();
            else if (messageText != null) messageText.text = "You Won!";
        }
    }
```
### GameManager.cs
```c++
using UnityEngine;
using TMPro;   // TextMeshPro namespace

public class GameManager : MonoBehaviour
{
    public static GameManager Instance;

    [Header("UI")]
    public TMP_Text messageText;     // TextMeshPro UI text
    public float messageDisplayTime = 0f;

    PlayerController player;

    void Awake()
    {
        if (Instance == null) Instance = this;
        else
        {
            Destroy(gameObject);
            return;
        }
    }

    void Start()
    {
        // Compatible with older Unity versions
        player = FindObjectOfType<PlayerController>();

        if (messageText != null)
            messageText.text = "";
    }

    public void GameOver()
    {
        Debug.Log("Game Over");

        if (messageText != null)
            messageText.text = "Game Over";

        if (player != null)
            player.enabled = false;

        if (messageDisplayTime > 0f)
            Invoke(nameof(ClearMessage), messageDisplayTime);
    }

    public void Win()
    {
        Debug.Log("You Won!");

        if (messageText != null)
            messageText.text = "You Won!";

        if (player != null)
            player.enabled = false;

        if (messageDisplayTime > 0f)
            Invoke(nameof(ClearMessage), messageDisplayTime);
    }

    void ClearMessage()
    {
        if (messageText != null)
            messageText.text = "";
    }

    public void RestartLevel()
    {
        UnityEngine.SceneManagement.SceneManager.LoadScene(
            UnityEngine.SceneManagement.SceneManager.GetActiveScene().buildIndex
        );
    }
}
```
## Output:
![WhatsApp Image 2025-11-20 at 20 03 19_443a0a46](https://github.com/user-attachments/assets/38935b80-5500-4214-987c-75d851e9ecfd)
![WhatsApp Image 2025-11-20 at 20 04 00_52e414ab](https://github.com/user-attachments/assets/cd57aaf0-c2ba-4d82-9c3a-222b189983ec)
![WhatsApp Image 2025-11-20 at 20 05 30_475e783b](https://github.com/user-attachments/assets/2b238179-3723-4ef6-a2cc-7d362c84be15)


## Result:
Thus the game was developed using Unity and adopted NavMesh technology.

