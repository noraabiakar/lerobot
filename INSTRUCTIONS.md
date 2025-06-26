1) Download Pi0 checkpoint from HuggingFace LeRobot
huggingface-cli download lerobot/pi0 --local-dir pi0

2) Setup the robot (calibrate)
- follow the robot calibration setup: https://huggingface.co/docs/lerobot/so101
  detect the motor buses: 
  
  python lerobot/find_port.py
   leader /dev/ttyACM1
   follower /dev/ttyACM0

  you will need to make the current user be able to use those devices
   (otherwise you would need to use sudo)
   cf. https://askubuntu.com/questions/133235/how-do-i-allow-non-root-access-to-ttyusb0
   if the devices are in the dialout group, then do

   sudo usermod -a -G dialout $USER

  calibrate the follower arm

python -m lerobot.calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=follower_arm 

  calibrate the leader arm

python -m lerobot.calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=leader_arm

Try to teleoperate (without camera)

python -m lerobot.teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=follower_arm \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \

    --teleop.id=leader_arm

3) Setup cameras

follow https://huggingface.co/docs/lerobot/cameras#setup-cameras

Identify the cameras:

python lerobot/find_cameras.py opencv

The video devices might not be accessible to the user.
Make the necessary changes

(cf. cf. https://askubuntu.com/questions/133235/how-do-i-allow-non-root-access-to-ttyusb0 )

sudo usermod -a -G video $USER

* setup the iphone camera: https://github.com/huggingface/lerobot/blob/b536f47e3ff8c3b340fc5efa52f0ece0a7212a57/examples/12_use_so101.md

Install the packages to enable virtual cameras:
sudo apt install v4l2loopback-dkms v4l-utils
If building the kernel module fails (as it does on Ubuntu 22.04 currently), you can install it from source: https://github.com/v4l2loopback/v4l2loopback.git

OBS studio

4) Start following the lego tutorial
https://huggingface.co/docs/lerobot/il_robots

# Recording dataset
python -m lerobot.record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM1 \
    --robot.id=follower_arm \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, side: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30}}" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM0 \
    --teleop.id=leader_arm \
    --display_data=true \
    --dataset.repo_id="noraabk/test0" \
    --dataset.root=/home/damien/git/ml/lerobot/goat_picking_2 \
    --dataset.num_episodes=100 \
    --dataset.push_to_hub=True \
    --dataset.single_task="Put the red goat toy in the bowl" \
    --dataset.episode_time_s=30 \
    --dataset.reset_time_s=20 \
    --resume=true

# Visualize a dataset
python lerobot/scripts/visualize_dataset_html.py --root ./goat_picking/  --repo-id noraabk/so101-goat-picking

# Training a model (ACT policy)
python -m lerobot.scripts.train \
    --policy.type=act \
    --policy.device=cuda \
    --output_dir=/storagte/models/act_so101_goat_50_v2 \
    --job_name=act_so101_goat \
    --dataset.repo_id=noraabk/test0 \
    --dataset.root=/storage/data/so101-goat-picking-v1 \
    --batch_size=32 \
    --steps=12500

# download base pi0 fast base model
huggingface-cli download lerobot/pi0fast_base --local-dir pi0fast_base

# Finetune a base model (Pi0 fast policy) (with LoRA)
pip install scipy

If issue with paligemma config, downgrade transformers version

python -m lerobot.scripts.train \
    --policy.path=/storage/models/pi0fast_base \
    --policy.device=cuda \
    --output_dir=models/pi0fast_so101_goat \
    --job_name=pi0fast_so101_goat \
    --dataset.repo_id=noraabk/test0 \
    --dataset.root=/home/nora/git/lerobot/goat_picking \
    --lora_config.rank=16 \
    --batch_size=4

# Evaluate the model
python -m lerobot.record  \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, side: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30}}" \
  --robot.id=follower_arm \
  --display_data=false \
  --dataset.repo_id="noraabk/eval_so101-goat-picking-v3" \
  --dataset.single_task="Put the red goat toy in the bowl" \
  --policy.path="/storage/models/"

# Evaluate the open pi models  
python -m lerobot.record  \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, side: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30}}" \
  --robot.id=follower_arm \
  --display_data=false \
  --dataset.repo_id="noraabk/eval_so101-goat-picking-v3" \
  --dataset.single_task="Put the red goat toy in the bowl" \
  --policy.path="/storage/models/openpi0fast_200_episodes_PI_impl"
