# Ex.No: 9  Implementation of Simple Reinforcement Learning 
### DATE:                                                                            
### REGISTER NUMBER : 212223230252
### AIM: 
To write a program to implement  Reinforcement learning  in Unity 
### Algorithm:
```
1.Create a new 3D Unity project
2.Create a plane → Right-click Hierarchy > 3D Object > Plane
3.Create an Agent (Cube)
4.3D Object → Cube → Rename to Agent
5. Add Rigidbody (disable gravity if needed)
6. Create a Target (Sphere)
7.3D Object → Sphere → Rename to Target
8.Create an empty GameObject → Area (to reset Agent and Target positions)
9.Add the Behavior Parameters component to your Agent
10.Behavior Name: MoveToTarget
11. Vector Observation: 7 (3 for agent pos + 3 for target pos + 1 for velocity), 
Action Space: Continuous (2)
12. run the command 
mlagents-learn config.yaml --run-id=move-to-target --train
```  
### Program:
```
using Unity.MLAgents;
using Unity.MLAgents.Actuators;
using Unity.MLAgents.Sensors;
using UnityEngine;

public class MoveToTargetAgent : Agent
{
    public Transform targetTransform;
    private Rigidbody agentRb;

    public override void Initialize()
    {
        agentRb = GetComponent<Rigidbody>();
    }

    public override void OnEpisodeBegin()
    {
        // Reset agent and target position
        agentRb.velocity = Vector3.zero;
        transform.localPosition = new Vector3(Random.Range(-4f, 4f), 0.5f, Random.Range(-4f, 4f));
        targetTransform.localPosition = new Vector3(Random.Range(-4f, 4f), 0.5f, Random.Range(-4f, 4f));
    }

    public override void CollectObservations(VectorSensor sensor)
    {
        // Positions
        sensor.AddObservation(transform.localPosition);
        sensor.AddObservation(targetTransform.localPosition);
        // Agent velocity
        sensor.AddObservation(agentRb.velocity);
    }

    public override void OnActionReceived(ActionBuffers actions)
    {
        Vector3 move = new Vector3(actions.ContinuousActions[0], 0, actions.ContinuousActions[1]);
        agentRb.AddForce(move * 10);

        // Reward
        float distanceToTarget = Vector3.Distance(transform.localPosition, targetTransform.localPosition);
        if (distanceToTarget < 1.2f)
        {
            SetReward(1.0f);
            EndEpisode();
        }

        // Fail condition
        if (transform.localPosition.y < 0)
        {
            EndEpisode();
        }
    }

    public override void Heuristic(in ActionBuffers actionsOut)
    {
        var cont = actionsOut.ContinuousActions;
        cont[0] = Input.GetAxis("Horizontal");
        cont[1] = Input.GetAxis("Vertical");
    }
}
config .yaml
behaviors:
  MoveToTarget:
    trainer_type: ppo
    max_steps: 50000
    time_horizon: 64
    summary_freq: 1000
    hyperparameters:
      batch_size: 64
      buffer_size: 2048
      learning_rate: 3.0e-4
      beta: 5.0e-4
      epsilon: 0.2
      lambd: 0.95
      num_epoch: 3
    network_settings:
      normalize: false
      hidden_units: 128
      num_layers: 2
    reward_signals:
      extrinsic:
        gamma: 0.99
        strength: 1.0
```
### Output:
### Unity Render:
<img width="1918" height="987" alt="image" src="https://github.com/user-attachments/assets/7e24bfe4-df0f-4e2d-a39c-22b681a06022" />

<img width="1918" height="1021" alt="image" src="https://github.com/user-attachments/assets/19170d5d-bbec-42cd-a8c1-fa4acb7a1a18" />

<img width="1918" height="997" alt="image" src="https://github.com/user-attachments/assets/38642eab-923b-4902-a500-8c73483ce426" />

### TensorBoard grapghs:

<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/69116d6e-73ae-4f76-9168-6db814cdbfc9" />

<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/dfb7e7bb-422a-455e-bc3c-a4a50971c78f" />

<img width="1920" height="1020" alt="image" src="https://github.com/user-attachments/assets/9748399b-b62c-47b6-a97b-959c20802101" />




### Result:
Thus the AI character was trained using reinforcement learning.
