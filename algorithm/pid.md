头文件

    struct pid_param
    {
      float p;
      float i;
      float d;
      float input_max_err;

      float max_out;
      float inte_limit;
    };

    typedef struct pid
    {
      struct pid_param param;

      float set;
      float get;

      float err;
      float last_err;

      float pout;
      float iout;
      float dout;
      float out;

      void (*f_param_init)(struct pid *pid,
                          float max_output,
                          float inte_limit,
                          float p,
                          float i,
                          float d);
      void (*f_pid_reset)(struct pid *pid, float p, float i, float d);
    }pid_t;

    void pid_struct_init(
        struct pid *pid,
        float maxout,
        float intergral_limit,

        float kp,
        float ki,
        float kd);

    float pid_calculate(struct pid *pid, float fdb, float ref);

C 文件

    #include "pid.h"
    void pid_abs_limit(float *a, float ABS_MAX)
    {
      if (*a > ABS_MAX)
        *a = ABS_MAX;
      if (*a < -ABS_MAX)
        *a = -ABS_MAX;
    }
    static void pid_param_init(
                  struct pid *pid,
                  float maxout,
                  float inte_limit,
                  float kp,
                  float ki,
                  float kd)
    {

      pid->param.inte_limit = inte_limit;
      pid->param.max_out = maxout;

      pid->param.p = kp;
      pid->param.i = ki;
      pid->param.d = kd;
    }

    /**
    * @代码运行时简短修改pid参数
    * @pid:控制pid结构
    * @参数[in] p/i/d: pid参数
    */
    static void pid_reset(struct pid *pid, float kp, float ki, float kd)
    {
      pid->param.p = kp;
      pid->param.i = ki;
      pid->param.d = kd;

      pid->pout = 0;
      pid->iout = 0;
      pid->dout = 0;
      pid->out = 0;
    }

    /**
    * @简单计算增量PID和位置PID
    * @param[in] pid:控制pid结构
    * @param[in] get:测量反馈值
    * @param[in] set:目标值
    * @retval pid计算输出
    */
    float pid_calculate(struct pid *pid, float get, float set)
    {
      pid->get = get;
      pid->set = set;
      pid->err = set - get;
      if ((pid->param.input_max_err != 0) && (fabs(pid->err) > pid->param.input_max_err))
      return 0;

      pid->pout = pid->param.p * pid->err;
      pid->iout += pid->param.i * pid->err;
      pid->dout = pid->param.d * (pid->err - pid->last_err);

      pid_abs_limit(&(pid->iout), pid->param.inte_limit);
      pid->out = pid->pout + pid->iout + pid->dout;
      pid_abs_limit(&(pid->out), pid->param.max_out);

      return pid->out;
    }
    /**
    * @简短初始化pid参数
    pid_struct_init(struct pid *pid,float maxout,float inte_limit,float kp,float ki,float kd)
    */
    void pid_struct_init(
        struct pid *pid,
        float maxout,
        float inte_limit,

        float kp,
        float ki,
        float kd)
    {
      pid->f_param_init = pid_param_init;
      pid->f_pid_reset = pid_reset;

      pid->f_param_init(pid, maxout, inte_limit, kp, ki, kd);
      pid->f_pid_reset(pid, kp, ki, kd);
    }

- 参数整定找最佳，从小到大顺序查；
- 先是比例后积分，最后再把微分加；
- 曲线振荡很频繁，比例度盘要放大；
- 曲线漂浮绕大湾，比例度盘往小扳；
- 曲线偏离回复慢，积分时间往下降；
- 曲线波动周期长，积分时间再加长；
- 曲线振荡频率快，先把微分降下来；
- 动差大来波动慢，微分时间应加长；
- 理想曲线两个波，前高后低 4 比 1。
